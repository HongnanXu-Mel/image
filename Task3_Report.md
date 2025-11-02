## 2. Implementation - Connectivity

### Cloud Services and Internet Data Integration

The Palate application extensively utilizes cloud services and internet-based data sources for a fully connected mobile experience.

**Firebase Integration**: Firebase serves as the backbone, providing authentication, database, and storage. Firebase Authentication handles secure email/password registration and login with "Remember Me" functionality providing 30-day persistence. Session management ensures users remain logged in across the app lifecycle.

**Firebase Authentication**: User registration and login use Firebase Auth with secure email/password authentication. The system maintains authentication state continuously and validates users before allowing data access.

**Firebase Firestore Database**: Firestore stores structured collections for reviews, users, restaurants, crowdFeedback, and comments. The application performs complex queries with filtering and ordering operations like `whereEqualTo("userId", userId)` and `orderBy("createdAt", Query.Direction.DESCENDING)`. Real-time data synchronization ensures immediate visibility of crowd feedback and review postings across all devices. Document updates use Firestore's update operations to maintain data consistency.

**Supabase Storage Integration**: Cloud-based image storage handles user profile pictures and review images. The SupabaseStorageService class, implemented in Kotlin, manages secure file uploads and downloads using signed URLs with one-year expiration. Integration with Kotlin coroutines demonstrates the application's multi-language approach combining Java for business logic and Kotlin for specialized services.

**Google Maps API Integration**: The Maps SDK provides interactive map functionality with restaurant location visualization and custom markers color-coded by crowd density status. Map controls include zoom in/out and my location. Camera animations and map state management enhance user experience. The Places API enables restaurant search with autocomplete functionality through PlacesClient.

**Location Services**: The application uses Fused Location Provider for accurate GPS positioning, combining GPS, Wi-Fi, and cellular network triangulation. The FusedLocationProviderClient provides location updates for the "My Location" feature.

**Network Communication Patterns**: All network calls are non-blocking and asynchronous using callback-based architecture. This ensures proper thread management with Android's main thread execution for UI updates while network operations complete in the background.

**Error Handling for Network Operations**: The application manages network timeouts, connection failures, and offline states. User-friendly error messages explain connectivity issues. Graceful degradation occurs when network access is unavailable.

**Real-Time Data Features**: Live updates support crowd density status changes, instant review visibility, and live vote counting. Dynamic user score updates respond to community activity in real-time.
