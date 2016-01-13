# Objective-c 

## Introduction

This document is useful if you work on a huge project and want to keep it readable, clean & maintainable.
Also helps you to avoid common mistakes (I have found since I started my career) and shows some best-practices.

## Class naming convention

Use class prefixes instead of just plain names. This makes your classes more unique - imagine if everybody would use plain names: you couldn't import a third party framework without conflict.
> You can set up the default prefix in Xcode: select the project file in the Navigator (file browser, left side),
> select File inspector tab and under Project Document / Class Prefix.

## Markers (pragma mark)

These markers can help your (and an other developer's) work to find methods.

Recommended markers in order:

* Initialization (-init, -dealloc, shared instance accessor of a singleton)

* Private methods

* View lifecycle (overrided methods of UIKit classes: -viewDidLoad, -viewWillAppear... etc.)

* Public methods

* Overrides

* Accessors

* Notification handlers

* KVO handler

* <[DelegateName]> (for delegate method implementations)

Usage:
`#pragma mark - [title]`

or embed markers:

`#pragma mark - [main title]`

`#pragma mark [sub-title]`

## Naming global constants

Just like notification names, enum types... etc. They need to reflect to the container class, so 
it should use it as a base. This help the user of this class to use code completion without checking the classes header.

Example:

	extern NSString * const MYClassDidSomethingNotification;

	extern const CGFloat MYClassConstantFloatValue;

	typedef NS_ENUM(NSInteger, MYClassSide)
	{
	MYClassSideLeft,
	MYClassSidesRight
	};

	@interface MyClass : ...


## Asserts

Use asserts to keep thigt conditions in DEBUG mode. It will help you to find programming mistakes and/or
invalid states in runtime.

Examples:

	- (BOOL)checkThis:(NSNumber *)number
	{
		NSParameterAssert(number);
		...
	}

	- (void)myMethod
	{
		const NSUInteger numberOfItems = [self numberOfItems];
		
		NSAssert(0 < numberOfItems, @"No item has been found");
		
		...
	}

Handy, isn't it?

> Avoid to use messages in the NSAssert body!
> The asserts are included when you're in DEBUG mode - so the message won't be sent in RELEASE build 
> which can change it's behaviour.

> Note: NSParameterAssert became deprecated when the _\_Nonnull_ annotation was announced.
> _- (void)checkThis:(NSNumber * \_Nonnull)number;_
> This would help you when compiling the app instead of getting exceptions in runtime, 
> but it's a real pain in the ass to add this to your existing codebase.

## Initializers

Always check the header for designated initializers! Override, or use it in your implementation.

> If you create a new one and your class does't work normally with the
> default initializer of it's superclass then mark it with `NS_DESIGNATED_INITIALIZER`

Example:

MYCar.h

	@interface MYCar : MYVehicle

	- (instancetype)initWithColor:(UIColor *)color numberOfWheels:(NSUInteger)numberOfWheels NS_DESIGNATED_INITIALIZER;
	...

	@end

MYCar.m

	@interface MYCar

	@property (nonatomic) NSUInteger numberOfWheels;

	@end


	@implementation

	- (instancetype)initWithColor:(UIColor * _Nonnull)color numberOfWheels:(NSUInteger)numberOfWheels
	{
		self = [super initWithColor:color];	// designated initializer of MYVehicle class
		if (self) {
			self.numberOfWheels = numberOfWheels;
		}

		return self;
	}

	...

	@end

## Properties

You need to keep in mind several things when using properties in your class.

### Visibility

If your property is public and read-only, make sure it always has the right value
(especially in UIViewController subclasses, see below).

### Weak, copy or strong?

In general: 

* Use weak for storing delegates,

* Copy for classes which have mutable versions (NSString, NSArray, NSDictionary...) and blocks,

* Strong for everything else.

### Local variable vs property

Local variables are slowly being dropped from the framework - I recommend to declare properties all the time
and use as _self.myProperty_ instead of it's local variable _\_myProperty_.

> Use _\_myPropertyName_ only in setters - especially after you overrode the default one!

## Constants vs dedicated methods

Let's see an example what I mean: there's an expression (or condition) I would like to use, 
but I'm not that lazy so I added an in-line comment:

	if (0 < [self.numberOfItems count]) {
		// Has items
		... 
	}

The inline comment is the 2nd worst you can use after the "using nothing".

It's much better if you name it, like *assign to a constant variable* in the original method:

	const BOOL hasItems = (0 < [self.numberOfItems count]);
	if (hasItems) { ... }

> Note: use brackets for complex expressions: the next line will be auto-indent when you break it. 

The best methods if the expression is *refactored to a method*, but I don't recommend if it's used 
just from only 1 place. Makes your implementation file longer and a bit more expensive to get the value.

	- (BOOL)hasItems
	{
		return (0 < [self.numberOfItems count]);
	}

## Singletons

You should create a singleton when your class shouldn't be instantiated more than once.

Some examples:

* Database Manager

* Music Player

* Account Manager

* Uploaders and Downloaders... (most of the network layer classes)
...

> Note: Watch out when subclassing a singleton: the shared instance getter class method won't 
> work in your subclass. You need to write the _+sharedInstance_ manually and 
> keep your (static) _sharedInstance_ in your implementation

If your singleton observes a property or reacts on a notification an initializer shoud be implemented which
must be called before the first use. Otherwise the notifications won't be handled in the singleton just after
the first access (which is not defined).

	static MYSingleton *sharedInstance = nil;
	+ (BOOL)isSharedIntanceInitialized
	{
		NSAssert(self == [MYThisActualClass class], @"Subclasses must implement this method!");
		
		return !!sharedInstance;
	}

	+ (void)initializeSharedInstance
	{
		NSAssert(self == [MYThisActualClass class], @"Subclasses must implement this method!");
		
		if ([self isSharedIntanceInitialized]) {
			// This point, the shared instance is already initialized and the assert will fire in DEBUG mode
			// But just to make sure, skip the re-initializing if the asserts are turned off.

			NSAssert(NO, @"Already initialized!");

			return;
		}

		sharedInstance = [[self alloc] init];
	}

	+ (instancetype)sharedInstance
	{
		NSAssert(self == [MYThisActualClass class], @"Subclasses must implement this method!");
		
		if (![self isSharedIntanceInitialized]) {
			// This point, the shared instance is hasn't been initialized and the assert will fire in DEBUG mode
			// But let's initialize the shared instance if the asserts are turned off.

			NSAssert(NO, @"Shared instance is not initialized but trying to access");

			[self initializeSharedInstance];
		}

		return sharedInstance;
	}

	- (id)init
	{
		self = [super init];
		if (self) {
			[[NSNotificationCenter defaultCenter] addObserver:self
			selector:@selector(handleOtherClassDidSomethingNotification:)
			name:MYOtherClassDidSomethingNotification
			object:nil];
		}

		return self;
	}

	// Not necessary in a singleton, but recommended
	- (void)dealloc
	{
		[[NSNotificationCenter defaultCenter] removeObserver:self];
	}
	...
	@end

## Notification handlers

Subscribe notifications for your instance but don't forget to unsubscribe. 
The last place you can do that is the _-dealloc_.

	- (id)init
	{
		self = [super init];
		if (self) {
			[[NSNotificationCenter defaultCenter] addObserver:self
			selector:@selector(handleOtherClassDidSomethingNotification:)
			name:MYOtherClassDidSomethingNotification
			object:nil];
		}

		return self;
	}

	- (void)dealloc
	{
		[[NSNotificationCenter defaultCenter] removeObserver:self];
	}

> Use dedicated methods for handling a notification! Avoid to bind methods like _-clearTableView_.
> The recommended naming is _-handle[notificatio_name_without_prefix]:_ and always add the 
> parameter - even if it's not in use.

## Key-value observers' handlers

Use the context parameter when subscribing for a KVO.

Example:

MYClass.m

	@interface MYClass ()
	{
		id otherObject;
	}

	@end

	static void *kvoContext = &kvoContext;

	@implementation MYClass

	#pragma mark - Initialization

	- (id)init
	{
		[self = super init];
		if (self) {
			otherObject = ...
			[otherObject addObserver:self 
			forKeyPath:@"parameter"
			options:(NSKeyValueObservingOptions)0
			context:kvoContext];
	}

		return self;
	}

	- (void)dealloc
	{
		[otherObject removeObserver:self forKeyPath:@"parameter" context:kvoContext];
	}

	#pragma mark - KVO

	- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
	{
		if (context != kvoContext) {
			// Not mine > send to the superclass & bail out
			[super observeValueForKeyPath:keyPath ofObject:object change:change context:context];

			return;
		}

		// Handle update
		...
	}

## Error handling

Objective-C using the old-school C-type error handling. If a method provides error, that does with an in&out parameter + a BOOL return value.

Example:
	- (BOOL)openFile:(NSError * __autoreleasing *)error;	// YES, if the file is opened successfully (and error is nil) or NO as return value plus an NSError instance as error.

How to implement?

Example:

	- (BOOL)openFile:(NSError * __autoreleasing *)error
	{
		BOOL success = NO;	// Will be the return value
		NSError * internalError;

		... // success flag & internal error can be set in here
		
		// Assign error value
		if (error) {	// We need to check it because it can be nil
			*error = [internalError copy];	// Watch out: *error = internalError; is not good enough!
		}

		return success;
	}

> Avoid to throw exception! The debugger will pause the running even you have a _@try_ around that point.

## Metadata vs model objects

Create objects from metadata as soon as possible!

Example: Getting response JSON from the server:

	[
		{
			id : 4
			title : "How to develop application for iPhone"
			pages : 358
		}, 
		...
	]

After the serialization we get an _NSArray_ with _NSDictionaries_.

Of course we use named constants for the keys like:

	static NSString * const MYDownloaderServerResponseKeyBookIdKey = @"id";
	static NSString * const MYDownloaderServerResponseKeyBookTitleKey = @"title";
	...

We could make these public and use the dictionary everywhere, but we have more clean code if use 
model object:

MYBook.h

	@interface MYBook : NSObject

	@property (nonatomic, readonly, strong) NSNumber *bookId;
	@property (nonatomic, readonly, strong) NSString *title;
	@property (nonatomic, readonly, strong) NSNumber *pages;

	+ (instancetype)bookWithId:(NSNumber *)bookId title:(NSString *)title pages:(NSNumber *)pages;
	// and / or 
	- (instancetype)initBookWithId:(NSNumber *)bookId title:(NSString *)title pages:(NSNumber *)pages;

	@end

## Macros < C-functions

[Macros are dangerous](http://stackoverflow.com/questions/14041453/why-are-preprocessor-macros-evil-and-what-are-the-alternatives) but those easily can be replaced with C functions.

	#define ADD(a, b) ...

Replace with:

	static NSInteger ADD(NSInteger a, NSInteger b) 
	{ ... }



## [Lambda expressions - alias "blocks"](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Blocks/index.html)

Use blocks instead of delegate callbacks for singleton classes.

> Avoid to retain self in block!

	__weak MYViewController *weakSelf = self;
	[[MYDownloader sharedInstance] start:^(NSArray<MYObject *> *result, NSError error) {
		if (error) {
			[weakSelf showErrorMessage:error];
		} else {
			[weakSelf updateContent:result];
		}
	}];

## Some useful UIKit patterns

### Properties of UIViewController

Let's see this example:

MYViewController.h

	@interface MYViewController : UIViewController

	@property (nonatomic, readonly, strong) UIView *backgroundView;

	@end

MYViewController.m

	@implementation MYViewController

	@syntesize backgroundView = _backgroundView;
	
	#pragma mark - Public methods

	- (void)viewDidLoad
	{
		[super viewDidLoad];

		UIView *backgroundView = self.backgroundView;
		backgroundView.frame = self.view.bounds;
		[self.view addSubview:backgroundView];
		...
	}

	#pragma mark - Public methods

	#pragma mark Accessors

	- (UIView *)backgroundView
	{
		if (!_backgroundView) {
			_backgroundView = [[UIView alloc] initWithFrame:CGRectZero];
			
			_backgroundView.backgroundColor = [UIColor blueColor];
		}
		
		return _backgroundView;
	}

	@end

> If the backgroundView property was initialized in the _-viewDidLoad_, it would be inaccessable before
> the first use of the view. If it happend in the _-initWithNibName:bundle:_ we would create a _UIView_
> instance and store in the memory without knowing it will be used or not.

### View lifecycle

In a UIView subclass, all the subviews should be created in the _-initWithFrame:_ initializer. 
If the view uses _NSLayoutConstraints_ to layout it's subviews, those should be set in _-layoutSubviews_.

### Nib/Xib/Storyboard vs pure code

There are 2 kinds of people: who think building in Interface Builder is a good thing and who don't.
I made several performance tests and the conclusion was the code-base UI is faster but slower to implement, 
harder to maintain (especially for junior developers) and require good overview to the user interface.
But I still prefer building the whole UI from code instead of drag&droping + coding.
