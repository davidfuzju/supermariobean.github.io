---
layout: post
category: iOS
---

new:

[networked-core-data-application](http://www.objc.io/issue-10/networked-core-data-application.html)

[sync-case-study](http://www.objc.io/issue-10/sync-case-study.html)

+ 验证core data

	所有的managed object都有一个fault状态，直到需要用到其属性的时候，才会将managed object的所有数据加载到内存，解除fault状态。而在此之前，也就是说在fault状态下，只会在内存中留有极小foot print的fault的状态。这就有一个问题，如何让已经加载到内存的managed object在我们不需要它的时候，再次进入fault状态，以减少内存的使用量。

	>实际上这里还涉及一个relationship之间的强引用而导致的循环引用问题，暂不讨论

	[apple doc](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreData/Articles/cdMemory.html#//apple_ref/doc/uid/TP40001860-SW1)提示我们可以调用

		refreshObject: mergeChanges:

	方法来解决这个问题。

	描述如下：

		You typically use refreshObject:mergeChanges: to refresh a managed object’s property
		values.

		If the mergeChanges flag is YES, the method merges the object’s property values with
		those of the object available in the persistent store coordinator.

		If the flag is NO, however, the method simply turns an object back into a fault
		without merging, which causes it to break strong references to related managed
		objects. This breaks the strong reference cycle between that managed object
		and the other managed objects.

	为了验证这个问题，通过项目[issue-4-full-core-data-application](https://github.com/SuperMarioBean/issue-4-full-core-data-application)实现

	分以下情况

	+ 记录没有持久化的情况下，针对创建出来的managed object调用

			[self.managedObjectContext refreshObject:item mergeChanges:NO];

		该条记录如果有nsfetchedresultcontroller而在tableview或者collectionview上显示的话，进入fault模式会直接以delete的方式从UI上面消除，与其有relationship 和 reverse relationship的managed object也会因为失去强引用而被释放（这里实际上就是打破了一个循环引用），即便调用save也无法保存其内容。

		并且再次通过的no fault操作（比如获取其中对象数据）时，会无法使其重新加载到内存中。

		可以理解为，如果在调用save或者本身就没有持久化的情况下，创建出来的对象只在内存中存在，如果调用refreshObject: mergeChanges: with flag NO，相当于直接从其拥有者手中丢弃该对象

		>默认的，managed Context不持有managed object，这里的拥有者主要指fetched object controller

		如果调用：

			[self.managedObjectContext refreshObject:item mergeChanges:YES];

		则会发生以下问题：

		>2014-02-25 11:01:16.367 NestedTodoList[2652:60b] *** Assertion failure in -[FetchedResultsControllerDataSource controller:didChangeObject:atIndexPath:forChangeType:newIndexPath:], /Users/DavidFu/Repositories/issue-4-full-core-data-application/NestedTodoList/FetchedResultsControllerDataSource.m:81
		>2014-02-25 11:01:16.369 NestedTodoList[2652:60b] CoreData: error: Serious application error.  Exception was caught during Core Data change processing.  This is usually a bug within an observer of NSManagedObjectContextObjectsDidChangeNotification.   with userInfo (null)
		>2014-02-25 11:01:16.371 NestedTodoList[2652:60b] *** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: ''

		第一行第三行是下面将会描述的一个问题, 实际上第二行的问题也因为上一个问题的解决而消失。

		log中显示，对象没有进入fault状态，重看注释：

			if flag is YES, merges an object with the state of the object available in
			the persistent store coordinator;

			if flag is NO, simply refaults an object without merging (which also causes
			other related managed objects to be released, so you can use this method to
			trim the portion of your object graph you want to hold in memory)

		在YES状态下，只会合并,不会fault对象，合并的意思是回合持久化存储中可用的对象在属性关系上的合并。

		根据描述可以大致猜测问题的原因可能是因为该对象没有在 persistent store coordinator有相对应的对象，所以无法利用可用的对象进行合并更新。

	+ 记录在有持久化的情况下，针对创建出来的managed object调用

			[self.managedObjectContext refreshObject:item mergeChanges:NO];

		不出所料的，保存了持久化对象之后，该方法只是简单地将一个内存中消耗客观内存的managed object的非fault对象转变为一个只有极小foot print的对象，并且在需要的时候重新载入内存。

		意外但是发生以下bug

		>Assertion failure in -[FetchedResultsControllerDataSource controller:didChangeObject:atIndexPath:forChangeType:newIndexPath:],/Users/DavidFu/Repositories/issue-4-full-core-data-application/NestedTodoList/FetchedResultsControllerDataSource.m:81
		>2014-02-25 11:16:51.239 NestedTodoList[2670:60b] *** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: ''

		>这个bug很意外的暴露了一个实现细节，经过排查，这个bug的引发原因是因为没有实现NSFetchedResultsController的NSFetchedResultsChangeUpdate枚举值，但是测试的步骤里面没有这更新记录这一步。显然，问题出在我们重新fault这次操作，在上面的的无持久化的情况下，如果fault一个已经显示在UI组件上面的managed object，NSFetchedResultsController会对delegate发送一个update的消息，导致该fault对象重新非fault。这对于没有持久化的情况来说，其实实际上就是发生了一次delete的消息。

		如果调用：

			[self.managedObjectContext refreshObject:item mergeChanges:YES];

		这不会有有任何改变

		stackoverflow上提问[What is the difference between managed object context save and refreshObject:mergeChanges:](http://stackoverflow.com/questions/15443070/what-is-the-difference-between-managed-object-context-save-and-refreshobjectmer/15443224#15443224)

		解释是

			[self.managedObjectContext refreshObject:item mergeChanges:YES];

		的作用是将

			-refreshObject:mergeChanges: does something quite different.
			It reads the current state of the object from the persistent store coordinator
			(which reads from the persistent store, and so on).

			Passing YES for mergeChanges means to keep any local modifications to the
			object intact and only update the fields that weren't changed. This is
			 pretty much the opposite of -save:.

		而

			[self.managedObjectContext save:nil]

		的作用则是

			-save: saves the changes you've made to any managed object in the context.
			This means they get flushed to the persistent store coordinator, which then
			writes them to the persistent store, which writes them to disk
			(assuming a disk-backed store).

		应用场景是为了 如果你知道managed object 从另外一个context中被改变，而且你希望在当前的context中看到这些改变。

		>这里提及了在多线程中使用不同的managed object context场景下的数据一致性问题

+ core data 中relationShip和 reverse relationShip之间引发的循环引用

+ multi NSManagedObjectContext app

## 引用

[How to deal with Core Data retain cycles](http://stackoverflow.com/questions/12624478/how-to-deal-with-core-data-retain-cycles)

[Should CoreData inverse relationships be represented as retained properties?](http://stackoverflow.com/questions/7946247/should-coredata-inverse-relationships-be-represented-as-retained-properties)

[core-data-memory-management](http://stackoverflow.com/questions/1734554/core-data-memory-management)

[Avoiding Ten Big Mistakes iOS Developers Make with Core Data](http://www.informit.com/articles/article.aspx?p=2160906)
