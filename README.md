// ==============================================================================
// Dart Basics Demonstration Program
// File: dart_basics_demo.dart
// Description: Comprehensive walkthrough of Dart language core fundamentals:
//              Types, Null Safety, Collections, Functions, OOP, and Async.
// ==============================================================================

import 'dart:async';

void main() async {
  print('====================================================');
  print('          WELCOME TO DART BASICS MASTERCLASS         ');
  print('====================================================\n');

  // --------------------------------------------------------------------------
  // 1. VARIABLES, DATA TYPES & TYPE INFERENCE
  // --------------------------------------------------------------------------
  print('--- 1. Variables & Types ---');
  int age = 22;
  double height = 5.9;
  String developerName = 'Basith';
  bool isFlutterDev = true;

  // Type inference using 'var' and immutable 'final' / 'const'
  var framework = 'Flutter'; // Type inferred as String
  final DateTime executionTime = DateTime.now(); // Runtime constant
  const double piValue = 3.14159; // Compile-time constant

  print('Developer: $developerName, Age: $age, Height: $height ft');
  print('Framework: $framework (Active: $isFlutterDev)');
  print('Executed at: $executionTime | Constant PI: $piValue\n');

  // --------------------------------------------------------------------------
  // 2. NULL SAFETY (Sound Null Safety in modern Dart 3+)
  // --------------------------------------------------------------------------
  print('--- 2. Sound Null Safety ---');
  String nonNullable = "Always has a value";
  String? nullableString; // Can be null

  print('Non-nullable: $nonNullable');
  print('Nullable before assignment: $nullableString');

  // Null-aware operator (??)
  String displayName = nullableString ?? 'Guest User';
  print('Fallback with ??: $displayName');

  // Null-aware assignment (??=)
  nullableString ??= 'Assigned Default Value';
  print('Nullable after ??=: $nullableString');

  // Null-aware access (?.)
  print('Length via ?.: ${nullableString?.length}\n');

  // --------------------------------------------------------------------------
  // 3. COLLECTIONS (List, Set, Map) & Modern Operators
  // --------------------------------------------------------------------------
  print('--- 3. Collections & Collection Operators ---');
  List<String> coreWidgets = ['Text', 'Container', 'Row', 'Column'];
  coreWidgets.add('Stack');

  Set<int> uniqueIds = {101, 102, 103, 101}; // Duplicate 101 is discarded
  Map<String, dynamic> appConfig = {
    'appName': 'FlutterResponsiveApp',
    'version': 1.0,
    'supportedPlatforms': ['Android', 'iOS', 'Web', 'Desktop']
  };

  // Collection if and Collection for with spread operator (...)
  bool includeDrawer = true;
  List<String> navItems = [
    'Home',
    'Profile',
    if (includeDrawer) 'Settings',
    for (var widget in coreWidgets.take(2)) 'Widget: $widget'
  ];

  print('Widgets List: $coreWidgets');
  print('Unique IDs Set: $uniqueIds');
  print('Config Map: $appConfig');
  print('Dynamic Navigation Items: $navItems\n');

  // --------------------------------------------------------------------------
  // 4. FUNCTIONS & HIGHER-ORDER METHODS
  // --------------------------------------------------------------------------
  print('--- 4. Functions & Functional Methods ---');
  greetUser(name: developerName, role: 'Software Engineer');

  // Higher-order function (.where, .map)
  List<int> numbers = [1, 2, 3, 4, 5, 6];
  var evenSquares = numbers
      .where((n) => n % 2 == 0)
      .map((n) => n * n)
      .toList();
  print('Original numbers: $numbers');
  print('Even squares: $evenSquares\n');

  // --------------------------------------------------------------------------
  // 5. OBJECT-ORIENTED PROGRAMMING (OOP)
  // --------------------------------------------------------------------------
  print('--- 5. Object-Oriented Programming (OOP) ---');
  final dev = MobileDeveloper(
    id: 'DEV-001',
    name: developerName,
    primaryLanguage: 'Dart',
    skills: ['Flutter', 'State Management', 'Responsive Design'],
  );

  dev.introduce();
  dev.writeCode();
  dev.logActivity('Built responsive layout structure');
  print('');

  // --------------------------------------------------------------------------
  // 6. ASYNCHRONOUS PROGRAMMING (Future, async, await)
  // --------------------------------------------------------------------------
  print('--- 6. Asynchronous Programming (Async/Await) ---');
  print('Fetching simulated API data...');
  final apiResponse = await fetchUserProfile(dev.id);
  print('API Response Received: $apiResponse\n');

  print('====================================================');
  print('         DART BASICS COMPLETED SUCCESSFULLY         ');
  print('====================================================');
}

// Named parameters with 'required' and default values
void greetUser({required String name, String role = 'Developer'}) {
  print('Hello, $name! Role assigned: $role');
}

// ----------------------------------------------------------------------------
// OOP: Abstract Class, Mixin, Class Inheritance
// ----------------------------------------------------------------------------

// Mixin for reusable behavior
mixin LoggerMixin {
  void logActivity(String action) {
    print('[AUDIT LOG ${DateTime.now().toIso8601String()}] Action: $action');
  }
}

// Abstract base class (Contract)
abstract class Person {
  final String id;
  final String name;

  Person({required this.id, required this.name});

  void introduce();
}

// Concrete class inheriting Person and applying LoggerMixin
class MobileDeveloper extends Person with LoggerMixin {
  final String primaryLanguage;
  final List<String> skills;

  MobileDeveloper({
    required super.id,
    required super.name,
    required this.primaryLanguage,
    required this.skills,
  });

  @override
  void introduce() {
    print('Hi, I am $name (ID: $id).');
    print('Specialization: $primaryLanguage Development.');
    print('Core competencies: ${skills.join(", ")}');
  }

  void writeCode() {
    print('$name is actively coding high-performance Flutter widgets!');
  }
}

// ----------------------------------------------------------------------------
// ASYNCHRONOUS FUNCTION
// ----------------------------------------------------------------------------
Future<Map<String, dynamic>> fetchUserProfile(String userId) async {
  // Simulating 1 second network latency
  await Future.delayed(const Duration(milliseconds: 600));
  return {
    'userId': userId,
    'status': 'Active',
    'reputationScore': 98.5,
    'lastLogin': DateTime.now().toIso8601String(),
  };
}
