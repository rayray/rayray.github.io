---
layout: singlepost
title: Tracking Core Data Migration Progress
---

Several months ago, I needed to perform a big Core Data migration in an iOS app. So big that it could take more than a couple minutes if the user made a lot of content with images.

Aside: This migration needed to happen because someone wanted to store the images in Core Data many years ago. Woops! Even if you check the `Allows External Storage` box in Xcode, interacting with stored blobs through Core Data is painfully slow.

## So I Wanted To Show Migration Progress

I wrote a class to handle the migration and pass along progress to a delegate. 

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

Ugh, that's a lot of code to just to get a progress update to show the user.

But it's worse than we thought. Once the migration is running, if we have `NSEntityMigrationPolicy`s for our objects, we will see 
that `migrationProgress` **stops incrementing** while our policies get applied to each object. 😭

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

And then in our `observeValue` override, we can figure out how to display that value to the user. It's useful enough to say "Hey, we've moved X out Y images!"
while the user waits. People love watching numbers go up.

If you see a way to improve this or something I missed, please let me know on [Twitter](https://twitter.com/raymondedwards).
Thanks!
