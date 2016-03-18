# case-study-Moya-Realm-ObjectMapper
A little case study about a project using Moya + Realm + ObjectMapper combined together

## What
Talk about how i used these tree awesome technologies to build an iOS project:
* [Moya](https://github.com/Moya/Moya)
* [Realm](https://realm.io/)
* [ObjectMapper](https://github.com/Hearst-DD/ObjectMapper)
I'm not going too much deep with the code, my goal is to describe principally how i combined these three elements together and which difficulties or benefits i found using them.

## Platform and programming language
iOS 9.2 - XCode 7.2.1 - Swift

## Glance to the project
An enterprise iOS app for internal use in our company. Back-end previously done with REST API. Data received and sent in JSON format.
Necessity to store all new downloaded data on device. Possibility to edit model's entities information and send them to the back-end.
I use [CocoaPods](https://cocoapods.org/) to install all the packages.

## Initialization

### Realm
My first impression after just 10 minutes is: AWSOME! Realm really replace SQLite and Core Data Mobile, easy to set and easy to use.
Basic setup in `AppDelegate`
```
//Realm BASIC migration setup
   let config = Realm.Configuration(
     // Set the new schema version. This must be greater than the previously used
     // version (if you've never set a schema version before, the version is 0).
     schemaVersion: 4,

     // Set the block which will be called automatically when opening a Realm with
     // a schema version lower than the one set above
     migrationBlock: { migration, oldSchemaVersion in
       // We haven’t migrated anything yet, so oldSchemaVersion == 0
       if (oldSchemaVersion < 3) {
         // Nothing to do!
         // Realm will automatically detect new properties and removed properties
         // And will update the schema on disk automatically
       }
   })

   // Tell Realm to use this new configuration object for the default Realm
   Realm.Configuration.defaultConfiguration = config

   // Now that we've told Realm how to handle the schema change, opening the file
   // will automatically perform the migration
   let realm = try! Realm()
   print(realm.path)
```
"And it works like a charm", now you can go and start using and modify your model where and when you want. Just remember to switch the `schemaVersion` and `oldSchemaVersion` each time.
I suggest you to always print `realm.path()`: it is useful to quickly identify Realm's database location in Finder and open it with [Realm Browser](https://itunes.apple.com/it/app/realm-browser/id1007457278?mt=12).
Ok, so here there's a little snippet about one of the classes implemented with Realm model.
```

class User: Object {

//MARK - Properties
  dynamic var id:Int = 0
  dynamic var email:String = ""
  ...
  var skills = List<Skill>()

  override static func primaryKey() -> String? {
    return "id"
  }
...  

```    
More information and settings can be found obviously in [Realm's docs](https://realm.io/docs/swift/latest/).
I spend some time on the correct setting of `skills`: since it is a `List` it can't be a dynamic var, so i resolved omitting its type.

### ObjectMapper
This framework is helping you converting model objects contained in your project to and from JSON.
Also in this case i didn't find much real difficulties. Below another code snippet.
```
import Foundation
import RealmSwift
import ObjectMapper

class User: Object, Mappable {

//MARK - Properties
  dynamic var id:Int = 0
  dynamic var email:String = ""
  ...
  var skills = List<Skill>()

  override static func primaryKey() -> String? {
    return "id"
  }

//MARK - Initializers
  required convenience init?(_ map: Map) {
    self.init()
  }

//MARK - Mapping
  func mapping(map: Map) {
    id <- map["id"]
    email <- map["email"]
    ...
    let realm = try! Realm()    
    var skillIdsArray = [Int]()
    skillIdsArray <- map["skill_ids"]
    for skillId in skillIdsArray {
      ...//Here there are some other instructions to verify if the skill was effectively in Realm database
      skills.append(skill)
    }
  }

}

```
As you can see not much code has been added to `User` class: it just to implement `Mappable` protocol and be careful with `required convenience init?(_ map: Map)` and with `func mapping(map: Map)`. ObjectMapper will automatically be able to to generate your Realm model.
The only thing i point out is about the model generation of `skills`: as you can see and as i told you before, it is a `List` of `Skill` entities. In my case i wanted to persist also these kind of objects but to do so you must iterate over the `Array` obtained from JSON and then append it to the `List`.

### Moya
So, here game become harder or anyway it took me a while to understand the Moya's pattern but once i got it everything works great!
I had to look and search very hard in all open and close issues in the [GitHub project](https://github.com/Moya/Moya/issues), because there are not much tutorials or guides out there which could help me. What i noticed is: for simple examples everybody gives you information but when and if you need something with an higher complexity you must get your hans dirty!
What i understand:
* **Targets**: the way how you represent and define your API
* **Provider**: entity which makes all API requests
* **Endpoints**: provider map targets to endpoints and with them they do real network requests

Then i go on with this logic:
* `MyProjectAPI` -> the target file that contains your API definition with relatives endpoints; request type and cases they have to be used (i.e. `.GET` or `.PUT`); endpoint closure with which i'm doing requests; possible request passed parameters to use with a certain API; other stuff you like to personalize your requests.
The base of `MyProjectAPI` is the targets definition, with an `ENUM`
```
public enum MyProjectAPI {
  case GetAllUsers
  case EditUser(user: AnyObject)
}
```
Each target has its base URL which is your API endpoint
```
public var path: String {
    switch self {
    case .GetAllUsers:
      return "/users"
    case .EditUser:
      return "/users/id"
    }
  }
```
On the back-end side, to let you understand better, this is one of my endpoints: `http://my.site.com/api/users`.
Targets could have different requests type
```
//Request type
  public var method: Moya.Method {
    switch self{
    case .EditUser:
      return .PUT
    default:
      return .GET
    }
  }
```
Targets could have necessity to pass some parameters while sending the request, in my case `EditUser(user: AnyObject)`
```
//Possible passed parameters
  public var parameters: [String: AnyObject]? {
    switch self {
    case .EditUser(let user):
      let currentUser: User = user as! User
      //Closure for skills id array
      let skills_ids = {() -> [Int] in
        var skills_ids = [Int]()
        for skill in currentUser.skills {
          let skill_id = (skill as Skill).id
          skills_ids.append(skill_id)
        }
        return skills_ids
      }
      return [
        "user" : [
          "id" : currentUser.id,
          "email" : currentUser.email,
          ...
          "skill_ids" : skills_ids()
        ]
      ]
    default:
      return nil
    }
  }
```
