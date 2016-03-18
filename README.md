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
