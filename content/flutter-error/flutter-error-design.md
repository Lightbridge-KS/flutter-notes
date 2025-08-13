# Error Handling Design

I'll help you design a robust error handling strategy for your Flutter application with a clean architectural approach.

## Error Handling Architecture Overview

Here's a conceptual diagram of how errors should flow through your application layers:

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   UI Layer      │    │  Service Layer   │    │  SDK/HTTP       │
│  (Widgets)      │    │  (Business Logic)│    │  (Data Source)  │
├─────────────────┤    ├──────────────────┤    ├─────────────────┤
│ • User feedback │◄───│ • Error mapping  │◄───│ • Network errors│
│ • Loading states│    │ • Retry logic    │    │ • API errors    │
│ • Error dialogs │    │ • Caching        │    │ • Timeouts      │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         ▲                        ▲                        ▲
         │                        │                        │
      Log UI                   Log Business            Log Technical
     Interactions              Operations               Details
```

## Error Categories & Handling Strategy

Let me break down the approach by layer:

### 1. SDK/HTTP Layer (Data Source)

**What to handle:**

- Network connectivity issues
- HTTP status codes (400, 401, 404, 500, etc.)
- Timeout errors
- Parsing/serialization errors

**What to log:**

```dart
class ApiException implements Exception {
  final int? statusCode;
  final String message;
  final String? endpoint;
  final DateTime timestamp;
  
  ApiException({
    this.statusCode,
    required this.message,
    this.endpoint,
  }) : timestamp = DateTime.now();
}

class HttpClient {
  Future<T> request<T>(String endpoint) async {
    try {
      final response = await http.get(Uri.parse(endpoint));
      
      // Log technical details
      logger.info('API Request: $endpoint - ${response.statusCode}');
      
      if (response.statusCode >= 400) {
        // Log error with full technical context
        logger.error(
          'API Error: $endpoint',
          error: 'Status: ${response.statusCode}, Body: ${response.body}',
        );
        
        throw ApiException(
          statusCode: response.statusCode,
          message: _mapHttpErrorMessage(response.statusCode),
          endpoint: endpoint,
        );
      }
      
      return _parseResponse<T>(response.body);
    } on SocketException {
      logger.error('Network connectivity error: $endpoint');
      throw ApiException(message: 'No internet connection');
    } on TimeoutException {
      logger.error('Request timeout: $endpoint');
      throw ApiException(message: 'Request timed out');
    }
  }
}
```

### 2. Service Layer (Business Logic)

**What to handle:**

- Transform technical errors into business-friendly errors
- Implement retry mechanisms
- Handle caching fallbacks
- Business rule validation

**What to log:**

```dart
abstract class AppException implements Exception {
  final String userMessage;
  final String? technicalMessage;
  final DateTime timestamp;
  
  AppException({
    required this.userMessage,
    this.technicalMessage,
  }) : timestamp = DateTime.now();
}

class NetworkException extends AppException {
  NetworkException({String? technical}) 
    : super(
        userMessage: 'Please check your internet connection',
        technicalMessage: technical,
      );
}

class DataService {
  Future<List<Patient>> getPatients() async {
    try {
      final patients = await _httpClient.request<List<Patient>>('/patients');
      
      // Log successful business operation
      logger.info('Successfully retrieved ${patients.length} patients');
      
      return patients;
    } on ApiException catch (e) {
      // Log business context
      logger.warning(
        'Failed to retrieve patients',
        error: 'API Error: ${e.statusCode} - ${e.message}',
      );
      
      // Transform to user-friendly error
      if (e.statusCode == 401) {
        throw AuthenticationException();
      } else if (e.statusCode == null) {
        throw NetworkException(technical: e.message);
      } else {
        throw ServerException(technical: e.message);
      }
    }
  }
}
```

### 3. UI Layer (Presentation)

**What to handle:**

- Display user-friendly error messages
- Show loading states
- Provide retry options
- Navigate based on error type

**What to log:**

```dart
class PatientListScreen extends StatefulWidget {
  @override
  _PatientListScreenState createState() => _PatientListScreenState();
}

class _PatientListScreenState extends State<PatientListScreen> {
  Future<void> _loadPatients() async {
    try {
      setState(() => _isLoading = true);
      
      final patients = await _dataService.getPatients();
      setState(() {
        _patients = patients;
        _error = null;
      });
      
      // Log user interaction success
      logger.info('User successfully loaded patient list');
      
    } on AppException catch (e) {
      // Log user-facing error
      logger.info('User encountered error: ${e.userMessage}');
      
      setState(() => _error = e.userMessage);
      
      // Show user-friendly error
      if (e is AuthenticationException) {
        Navigator.pushReplacementNamed(context, '/login');
      } else {
        _showErrorSnackbar(e.userMessage);
      }
    } finally {
      setState(() => _isLoading = false);
    }
  }
}
```

## Logging Strategy

### What to Log at Each Layer:

**SDK Layer:**

- All HTTP requests/responses (status codes, endpoints)
- Network failures with technical details
- Parsing errors with data structure info

**Service Layer:**

- Business operation outcomes (success/failure counts)
- Data transformation issues
- Cache hits/misses
- Retry attempts

**UI Layer:**

- User interaction patterns
- Error presentation to users
- Navigation flows triggered by errors

### Sample Logging Setup:

```dart
import 'package:logger/logger.dart';

class AppLogger {
  static final Logger _logger = Logger(
    printer: PrettyPrinter(
      methodCount: 2,
      errorMethodCount: 8,
      lineLength: 120,
      colors: true,
      printEmojis: true,
    ),
  );
  
  static void info(String message, {dynamic error}) {
    _logger.i(message, error: error);
  }
  
  static void warning(String message, {dynamic error}) {
    _logger.w(message, error: error);
  }
  
  static void error(String message, {dynamic error, StackTrace? stackTrace}) {
    _logger.e(message, error: error, stackTrace: stackTrace);
  }
}
```

## Key Principles for Your Medical App Context:

**Patient Data Safety:**

- Never log sensitive patient information
- Use anonymized identifiers in logs
- Implement log retention policies

**User Experience:**

- Provide clear, non-technical error messages
- Offer actionable solutions ("Try again", "Check connection")
- Handle authentication errors gracefully

**Monitoring:**

- Track error patterns for system health
- Monitor API performance metrics
- Set up alerts for critical failures

This layered approach ensures technical details stay in logs while users see helpful messages, making your radiology application both robust and user-friendly.