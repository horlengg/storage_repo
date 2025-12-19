

![humnail.png](flutter-dev-mistack.png)


<br>


# Mistakes Most Flutter Developers Make

I've been involved in mobile development with Flutter, and through my experience, I've noticed that many developers—including some senior Flutter developers—make certain mistakes. While I’m not yet a senior developer myself, I wanted to write this article to share insights and corrections based on my research and testing. My goal is to help the community improve and avoid common pitfalls.


<br>

## UI Rebuilding

I've observed that many developers—including those working in professional solution companies—make mistakes in Flutter, especially when it comes to widget rebuilding. Often, they write everything as function-returned widgets inside a single state, which can cause the app to rebuild unnecessarily even for small changes. This leads to performance issues and a less efficient app.

***❌ Wrong***

```dart

class HomeView extends StatefulWidget {
  const HomeView({super.key});
  @override
  State<HomeView> createState() => _HomeViewState();
}

class _HomeViewState extends State<HomeView> {

  DateTime _currentTime = DateTime.now();
  Timer? _timeScheduler;

  @override
  void initState() {
    super.initState();
    _timeScheduler = Timer.periodic(Duration(seconds: 1),(timer) {
      setState(() {
        _currentTime = DateTime.now();
      });
    });
  }

  @override
  void dispose() {
    _timeScheduler?.cancel();
    super.dispose();
  }
  @override
  Widget build(BuildContext context) {
    log("HomeView rebuild()...");
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(20.0),
          child: Column(
            children: [
              // build time display
              _buildTimeWidget(),
              // build UI
              _buildAppUI()
              // ...
            ],
          ),
        ),
      ),
    );
  }
  Widget _buildTimeWidget(){
    log("HomeView._buildTimeWidget rebuild()...");
    return Text(
      "Current time : ${_currentTime
                  .toIso8601String()
                    .split('.')
                      .first}",
      style: TextStyle(color: Colors.green),
    );
  }
  Widget _buildAppUI(){
    log("HomeView._buildAppUI rebuild()...");
    return Container(
      child: null, //...
    );
  }
}
```

<br>

***✅ Right***

```dart


class HomeView extends StatefulWidget {
  const HomeView({super.key});
  @override
  State<HomeView> createState() => _HomeViewState();
}

class _HomeViewState extends State<HomeView> {
  
  @override
  Widget build(BuildContext context) {
    log("HomeView rebuild()...");
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(20.0),
          child: Column(
            children: [
              // build time display
              HomeTimeView(),
              // build UI
              _buildAppUI()
              // ...
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildAppUI(){
    log("HomeView._buildAppUI rebuild()...");
    return Container(
      child: null, //...
    );
  }
}

class HomeTimeView extends StatefulWidget {
  const HomeTimeView({super.key});

  @override
  State<HomeTimeView> createState() => _HomeTimeViewState();
}

class _HomeTimeViewState extends State<HomeTimeView> {

  DateTime _currentTime = DateTime.now();
  Timer? _timeScheduler;

  @override
  void initState() {
    super.initState();
    _timeScheduler = Timer.periodic(Duration(seconds: 1),(timer) {
      setState(() {
        _currentTime = DateTime.now();
      });
    });
  }

  @override
  void dispose() {
    _timeScheduler?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    log("HomeTimeView rebuild()...");
    return Text(
      "Current time : ${_currentTime
                  .toIso8601String()
                    .split('.')
                      .first}",
      style: TextStyle(color: Colors.green),
    );
  }
}

```

After separating the state, you'll notice that when the time changes, there's no need to rebuild the entire HomeView widget. Instead, only the HomeTimeView rebuilds, which helps prevent unnecessary rebuilds of the whole device UI and improves performance.

<br>

## Render Unnecessary Widget

#### 1. Simplifying Conditional Widget Rendering in Flutter

***❌ Wrong***

```dart

String? _errorMessage;

// inside widget tree
_errorMessage == null ? 
    Container() : 
    Text(_errorMessage!)

```
This approach uses a ternary operator to decide between rendering an empty **Container()** or a Text widget. While it works, it introduces an unnecessary widget (Container()) when there's no message to display. This can lead to extra widget tree complexity and minor performance overhead.

<br>

***✅ Right***

```dart

String? _errorMessage;

// inside widget tree
if(_errorMessage != null)
    Text(_errorMessage!)

```

Why it's better:

Using an if statement directly in the widget tree (with the spread operator if needed) is clearer and more idiomatic in Flutter. It renders the Text widget only when there's a message, avoiding unnecessary widgets like **Container()**. This makes the widget tree cleaner and easier to read.


#### 2. Avoiding Nested Widgets with Spread Operator in Flutter

***❌ Wrong***

```dart
Column(
    children: [
        if(_showInstruction)
            Column(
                children: [
                    Text("First widget..."),
                    Text("Second widget..."),
                    Text("Third widget..."),
                ],
            )
    ],
),
```

Here, you conditionally include a nested Column inside another Column. When _showInstruction is true, this creates a nested column, which might be unnecessary and could lead to layout issues or unnecessary widget nesting.

<br>

***✅ Right***

```dart

Column(
    children: [
        if(_showInstruction)
            ...[
                Text("First widget..."),
                Text("Second widget..."),
                Text("Third widget..."),
            ]
    ],
),

```

Why it's better:

Using the spread operator (...[]) allows you to insert multiple widgets directly into the parent Column. This flattens the widget tree, avoids nested Columns, and makes your code more concise and readable. It also aligns with Flutter's declarative style, making conditional rendering of multiple widgets clearer.

<br>
<br>

## Avoid Condition Conflict
I've see some developer write a easy to make it as unreadable and multiple line so I don't know what they are doing to make it harder and hander especialy for fresh developer that come to maintainent for add new feature.

<br>

***❌ Wrong***

```dart

if(
    trxType == 'MTI' ||
    trxType == 'MTO' ||
    trxType == 'RTI' ||
    trxType == 'MTP' ||
    trxType == 'MTL' ||
    trxType == 'IRT' ||
    trxType == 'RRI'
)
    Text("Transaction need approval")

```
This implementation is not a good practice, especially for maintainability and readability. As the list of transaction types grows, this approach becomes cumbersome and error-prone. 

<br>

***✅ Right***

```dart
/// Function check transaction by code whether transaction is need approval or not
bool isTrxNeedApproval(String trxType){
    return ['MTI','MTO','RTI','MTP','MTL','IRT','RRI'].contains(trxType);   
}

if(isTrxNeedApproval(trxType))
    Text("Transaction need approval")

```
This implementation is clean, maintainable, and scalable. By encapsulating the logic within a function, it enhances readability and makes it easy to update the list of transaction types that require approval in the future. Using a list's contains method provides a concise and efficient way to perform this check.

<br>

***✅ Advanced***

```dart
enum TrxType {
  bankTransfer('BT'),          
  internalTransfer('IT'),       
  requestTransferIn('RTI'),
  moneyTransferPending('MTP'),
  moneyTransferLocal('MTL'),
  internalRequestTransfer('IRT'),
  requestRefundIn('RFI'); 
  // ...

  final String code;
  const TrxType(this.code);

  /// Checks if the transaction needs approval
  bool get needsApproval => {
    bankTransfer,
    internalTransfer,
    requestTransferIn,
    moneyTransferPending,
    moneyTransferLocal,
    internalRequestTransfer,
    requestRefundIn,
  }.contains(this);

  /// Factory method to get enum from code
  static TrxType? fromCode(String code) {
    return TrxType.values.firstWhere(
      (type) => type.code == code,
      orElse: () => null,
    );
  }
}

// Usage example
final trxType = TrxType.fromCode(response.trxType);

// Inside widget tree
if (trxType?.needsApproval ?? false)
  Text("Transaction needs approval");

```

This approach is more advanced and improves code clarity by:

- Using enums to define transaction types with meaningful names.
- Encapsulating logic such as "needs approval" within the enum.
- Providing a method to convert from API response codes to enum instances.
- Enhancing readability and maintainability for developers.
- It makes the code self-explanatory and easier to extend or modify in the future.


