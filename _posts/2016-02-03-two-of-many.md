---
layout: singlepost
title: Tracking Migration Progress with NSMigrationManager and NSEntityMigrationPolicy
---

Several months ago, I needed to perform a manual Core Data migration in an iOS app. Because the model
stored images as binary data, we found our previous automatic migrations could take several minutes if
there were, say, 200 images stored in 200 objects. Sounds like a job for Improved UX!

(There's a side lesson here: don't store images in Core Data. I'll expand on this a later post.)

## So You Want To Show Migration Progress

I wrote a class to handle the migration and pass along info about progress. 

    // updated for Swift 3
    class MigrationManager: NSObject {
        
        weak var delegate: MigrationManagerDelegate?
        
        func migrate(from sourceURL: URL, to destinationURL: URL) throws {
            
            guard let sourceMetadata = try? NSPersistentStoreCoordinator.metadataForPersistentStore(ofType: NSSQLiteStoreType, at: sourceURL) else {
                throw MigrationError.noMetadata
            }
            
            guard let sourceModel = NSManagedObjectModel.mergedModel(from: [Bundle.main], forStoreMetadata: sourceMetadata) else {
                throw MigrationError.noSourceModel
            }
            
            // ...
            // get destination model
            // for more code surrounding this portion, you should reference the great objc.io article here:
            // https://www.objc.io/issues/4-core-data/core-data-migration/
            // ...
            
            let manager = NSMigrationManager(sourceModel: sourceModel, destinationModel: destinationModel)
            
            manager.addObserver(self, forKeyPath: "migrationProgress", options: .new, context: nil)

            // start migrating...
        }
        
        // ...

        override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
            if keyPath == "migrationProgress" {
                guard let manager = object as? NSMigrationManager else {
                    return
                }
                delegate?.migrationManager(manager: self, progress: manager.migrationProgress, userInfo: nil)
            }
        }
    }

Ugh, that's a lot of code to just to get a progress update.

But it's even worse than we thought. If we have `NSEntityMigrationPolicy`s for our objects, we will see 
that `migrationProgress` **stops incrementing** while our custom policies get executed for each object. 😭

However, since we're already customizing the migration of individual objects, we can get the 
total count of those objects the first time our policy is called and update the `userInfo` dictionary 
on the `NSMigrationManager` object.

    class Policy: NSEntityMigrationPolicy {
        override func createDestinationInstances(forSource sInstance: NSManagedObject, in mapping: NSEntityMapping, manager: NSMigrationManager) throws {
            
            if (manager.userInfo == nil) {
                let fetchRequest = NSFetchRequest<NSFetchRequestResult>(entityName: "MyObject")
                fetchRequest.includesSubentities = false
                
                // get the count and save it
                if let count = try? manager.sourceContext.count(for: fetchRequest),
                    count != NSNotFound
                {
                    manager.userInfo = ["totalCount": count]
                }
            }
            
            // ... set up our destination object
            
            // when we've migrated the object, let's give the current count
            if let userInfo = manager.userInfo {
                var mutableUserInfo = userInfo
                let currentCount = mutableUserInfo["currentCount"] as? Int ?? 0
                mutableUserInfo["currentCount"] = currentCount + 1
                manager.userInfo = mutableUserInfo
            }
        }
    }

Now that we've added a useful `userInfo` dictionary to the migration manager, we can observe changes back
in our `MigrationManager` container class.

    manager.addObserver(self, forKeyPath: "userInfo", options: .new, context: nil)

And then in our `observeValue` override, we can figure out how to display that value to the user. Since
I was dealing with user-created images, it was useful enough to say "Hey, we've moved X out Y images!"
while the user waited. People love watching numbers go up.

This solution at least gives us further insight into progress for objects that are migrated with a
custom policy. Given the tight time constraints when I wrote this, I was happy to just be able to combine the
object count with the progress value that `NSMigrationManager` gave me. (There may be a better way!)

If you see an obvious way to improve this or if you think I'm an idiot, please let me know on [Twitter](https://twitter.com/raymondedwards).
Thanks!
