---
layout: singlepost
title: Tracking Core Data Migration Progress
---

Some text.

## 2 header

Some more text.

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

Text again.

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

If you see a way to improve this or something I missed, please let me know on [Twitter](https://twitter.com/ray_at_work).
Thanks!
