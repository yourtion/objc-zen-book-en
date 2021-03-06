# Protocols

A great miss in the Objective-C world is the outcome of the latest decades about abstract interfaces. The term interface is typically used to refer to the `.h` file of a class but it has also another meaning well known to Java programmers which is basically used to describe to a set of methods not backed by a concrete implementation.
In Objective-C the latter case is achieved by using protocols. For historical reasons, protocols (used as Java interfaces) are not so much used in Objective-C code and in general by the community. It's mainly because the majority of the code developed at Apple don't embrace this approach and almost all developers tend to follow Apple's patterns and guidelines. Apple uses protocols almost only for the delegation pattern.
The concept of abstract interface is very powerful, has roots in the computer science history and there are no reasons to pretend it can't be used in Objective-C.

Here will be explained a powerful usage of protocols (intended as abstract interfaces) going through a concrete example: starting from a very bad designed architecture up to reaching a very well and reusable piece of code.

The example presented is the implementation of a RSS feed reader (think of it as a common test task asked during technical interviews).

The requirement is straightforward: presenting a remote RSS feed in a table view.

A very naïve approach would be to create a `UITableViewController` subclass and code the entire logic for the retrieving of the feed data, the parsing and the displaying in one place, which is, in other words, a MVC (Massive View Controller). This would work but it's very poorly designed and unfortunately it'd suffice to pass the interview step in some not-so-demanding tech startups.

A minimal step forward would be to follow the Single Responsibility Principle and create at least 2 components to do the different tasks:

- a feed parser to parse the results gathered from an endpoint
- a feed reader to display the results

The interfaces for these classes could be as so:

```obj-c

@interface ZOCFeedParser : NSObject

@property (nonatomic, weak) id <ZOCFeedParserDelegate> delegate;
@property (nonatomic, strong) NSURL *url;

- (id)initWithURL:(NSURL *)url;

- (BOOL)start;
- (void)stop;

@end

```

```obj-c

@interface ZOCTableViewController : UITableViewController

- (instancetype)initWithFeedParser:(ZOCFeedParser *)feedParser;

@end

```

The `ZOCFeedParser` is initialized with a `NSURL` to the endpoint to fetch the RSS feed (under the hood it will probably use NSXMLParser and NSXMLParserDelegate to create meaningful data) and the `ZOCTableViewController` is initialized with the parser. We want it to display the values retrieved by the parser and we do it using delegation with the following protocol:

```obj-c

@protocol ZOCFeedParserDelegate <NSObject>
@optional
- (void)feedParserDidStart:(ZOCFeedParser *)parser;
- (void)feedParser:(ZOCFeedParser *)parser didParseFeedInfo:(ZOCFeedInfoDTO *)info;
- (void)feedParser:(ZOCFeedParser *)parser didParseFeedItem:(ZOCFeedItemDTO *)item;
- (void)feedParserDidFinish:(ZOCFeedParser *)parser;
- (void)feedParser:(ZOCFeedParser *)parser didFailWithError:(NSError *)error;
@end

```

It's a perfectly reasonable and suitable protocol to deal with RSS, I'd say. The view controller will conform to it in the public interface:

```obj-c
@interface ZOCTableViewController : UITableViewController <ZOCFeedParserDelegate>
```

and the final creation code is like so: 

```obj-c
NSURL *feedURL = [NSURL URLWithString:@"http://bbc.co.uk/feed.rss"];

ZOCFeedParser *feedParser = [[ZOCFeedParser alloc] initWithURL:feedURL];

ZOCTableViewController *tableViewController = [[ZOCTableViewController alloc] initWithFeedParser:feedParser];
feedParser.delegate = tableViewController;
```

So far so good and you're probably happy with this new code, but how much of this code can be effectively reused? The view controller can only deal with objects of type `ZOCFeedParser`: at this point we just split the code in 2 components without any additional and tangible value than separation of responsibilities.

The responsibility of the view controller should be to "display items provided by <someone>" but if we are allowed to pass to it only `ZOCFeedParser` objects this can't hold. Here surfaces the need for a more general type of object to be used by the view controller.

We modify our feed parser introducing the `ZOCFeedParserProtocol` protocol (in the ZOCFeedParserProtocol.h file where also `ZOCFeedParserDelegate` will be).

```obj-c

@protocol ZOCFeedParserProtocol <NSObject>

@property (nonatomic, weak) id <ZOCFeedParserDelegate> delegate;
@property (nonatomic, strong) NSURL *url;

- (BOOL)start;
- (void)stop;

@end

@protocol ZOCFeedParserDelegate <NSObject>
@optional
- (void)feedParserDidStart:(id<ZOCFeedParserProtocol>)parser;
- (void)feedParser:(id<ZOCFeedParserProtocol>)parser didParseFeedInfo:(ZOCFeedInfoDTO *)info;
- (void)feedParser:(id<ZOCFeedParserProtocol>)parser didParseFeedItem:(ZOCFeedItemDTO *)item;
- (void)feedParserDidFinish:(id<ZOCFeedParserProtocol>)parser;
- (void)feedParser:(id<ZOCFeedParserProtocol>)parser didFailWithError:(NSError *)error;
@end

```

Notice that the delegate protocol now deals with objects conforming to our new protocol and the interface file of the ZOCFeedParser would be more skinny:

```obj-c

@interface ZOCFeedParser : NSObject <ZOCFeedParserProtocol>

- (id)initWithURL:(NSURL *)url;

@end

```

As `ZOCFeedParser`now conforms to `ZOCFeedParserProtocol`, it must implement all the required methods.
At this point the view controller can accept any object conforming to the new protocol, having the certaincy that the object will respond to `start` and `stop` methods and that it will provide info through the delegate property. This is all the view controller should know about the given objects and no implementation details should concern it.


```obj-c

@interface ZOCTableViewController : UITableViewController <ZOCFeedParserDelegate>

- (instancetype)initWithFeedParser:(id<ZOCFeedParserProtocol>)feedParser;

@end

```

The change in the above snippet of code might seem a minor change, but actually it is a huge improvement as the view controller would work against a contract rather than a concrete implementation. This leads to many advantages:

- the view controller can now accept any object that provide some information via the delegate property: this can be a RSS remote feed parser, a local one, a service that read other types of data remotely or even a service that fetch data from the local database;
- the feed parser object can be totally reused (as it was before after the first refactoring step);
- `ZOCFeedParser` and `ZOCFeedParserDelegate` can be reused by other components;
- `ZOCViewController` (UI logic apart) can be reused;
- it is easier to test as it'd be possible to use a mock object conforming to the expected protocol.

When implementing a protocol you should always strive to adhere to the [Liskov Substitution Principle](http://en.wikipedia.org/wiki/Liskov_substitution_principle). This principle states that you should be able to replace one implementation of an interface ("protocol" in the Objective-C lingo) with another without breaking either the client or the implementation.

In other words this means that your protocol should not leak the detail of the implementing classes; be even more careful when designing the abstraction expressed by you protocol and always keep in mind that the underlay implementation is irrelevant, what really matters is the contract that the abstraction expose to the consumer.

Everything that is designed to be reused in future is implicitly better code and should be always the programmer's goal. Code designed this way is what makes the difference between an experienced programmer and a newbie.

The final code proposed here can be found [here](http://github.com/albertodebortoli/ADBFeedReader).
