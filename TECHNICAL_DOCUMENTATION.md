# SavorApp - Complete Technical Documentation

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Technology Stack](#technology-stack)
3. [Architecture Overview](#architecture-overview)
4. [iOS App Architecture](#ios-app-architecture)
5. [Backend Integration](#backend-integration)
6. [Backend Architecture](#backend-architecture)
7. [Backend API Endpoints](#backend-api-endpoints)
8. [Backend External Integrations](#backend-external-integrations)
9. [Backend Database Schema](#backend-database-schema)
10. [Backend Configuration](#backend-configuration)
11. [Backend Error Handling](#backend-error-handling)
12. [Backend Deployment](#backend-deployment)
13. [RevenueCat Implementation](#revenuecat-implementation)
14. [Data Models](#data-models)
15. [Development Setup](#development-setup)

---

## Executive Summary

SavorApp (formerly VideotoRecipe) is a comprehensive iOS recipe management application that transforms cooking videos from social media platforms into structured, actionable recipes. The app combines AI-powered recipe extraction, intelligent meal planning, shopping list management, and ingredient analytics to create a complete cooking companion experience.

### Primary Purpose

- Extract structured recipes from video URLs (YouTube, TikTok, Instagram, Facebook)
- Generate recipes through conversational AI chatbot interface
- Plan meals for the week with AI assistance
- Manage shopping lists with Instacart integration
- Track ingredient usage and cooking patterns
- Provide subscription-based premium features via RevenueCat

### Key Features

- **Multi-source Recipe Extraction**: Video descriptions, captions, audio transcription, and OCR fallback
- **AI-Powered Processing**: OpenAI Whisper for transcription, GPT-4o for recipe extraction
- **Conversational Recipe Creation**: Chat-based recipe generation with voice input support
- **Meal Planning**: AI-generated meal plans with dietary preference filtering
- **Shopping Integration**: Instacart API support for shopping list creation
- **Subscription Management**: RevenueCat integration for free and pro tiers
- **Local Persistence**: SwiftData for offline recipe storage and management

---

## Technology Stack

### iOS App Stack

#### Core Framework & Language
- **Swift 5.9+**: Primary programming language
- **SwiftUI**: Declarative UI framework for all views
- **UIKit**: Used for tab bar styling, navigation bar appearance, and share extension UI
- **SwiftData**: Built-in persistence framework for local data storage

#### Third-Party SDKs
- **RevenueCat SDK**: Subscription management and in-app purchase handling
- **RevenueCatUI**: Pre-built paywall components

#### Apple Frameworks
- **Combine**: Reactive programming for state management and data flow
- **AVFoundation**: Audio/video processing for voice input and video playback
- **UserNotifications**: Push notifications for background task completion
- **Foundation**: Core utilities, networking, and data structures

#### Development Tools
- **Xcode**: Primary IDE and development environment
- **Swift Package Manager**: Dependency management (RevenueCat)
- **Git**: Version control

### Backend Stack

#### Backend Framework
- **FastAPI** (Python 3.11+): Main backend API server
  - Async/await support for concurrent operations
  - Automatic OpenAPI documentation
  - Pydantic models for data validation
  - Server-Sent Events (SSE) for streaming responses

#### Frontend API Layer
- **Next.js 14+** (TypeScript): Additional API routes
  - App Router with route handlers
  - Type-safe API with Zod validation
  - Server-side rendering support

#### Database
- **PostgreSQL** (Supabase): Recipe caching and storage
  - `asyncpg` for async database operations
  - JSONB for flexible recipe data storage
  - Connection pooling for performance

#### AI Services
- **OpenAI API**:
  - Whisper-1: Audio transcription
  - GPT-4o: Recipe extraction and structuring
  - GPT-4 Vision: OCR fallback for video frames
- **Google Gemini Vision API**: Alternative OCR for recipe text extraction

#### External APIs
- **Instacart Developer Platform**: Recipe page creation, store availability
- **Kroger Product API**: Product images via OAuth2
- **Amazon Product Advertising API**: Product search (optional)
- **Serper.dev**: Google Images search for ingredient photos
- **USDA FoodData Central**: Nutrition database

#### Video & Image Processing
- **yt-dlp**: Video/audio download from multiple platforms
- **OpenCV**: Video frame extraction
- **Pillow (PIL)**: Image processing
- **FFmpeg**: Video frame extraction (via subprocess)

#### Additional Libraries
- **httpx**: Async HTTP client
- **requests**: Synchronous HTTP requests
- **beautifulsoup4**: HTML parsing for video metadata
- **python-dotenv**: Environment variable management

---

## Architecture Overview

### System Architecture

```mermaid
graph TB
    iOSApp[iOS App<br/>SwiftUI + SwiftData]
    ShareExt[Share Extensions<br/>RecipeShareExtension<br/>VideoShareExtension]
    BackendAPI[Backend API<br/>FastAPI on Render.com]
    RevenueCat[RevenueCat<br/>Subscription Service]
    StoreKit[Apple StoreKit<br/>In-App Purchases]
    
    ExternalAPIs[External APIs<br/>OpenAI, Instacart,<br/>Kroger, USDA, etc.]
    
    iOSApp -->|HTTP Requests| BackendAPI
    iOSApp -->|Subscription Management| RevenueCat
    RevenueCat -->|Purchase Processing| StoreKit
    BackendAPI -->|AI Processing| ExternalAPIs
    ShareExt -->|Share Video URLs| iOSApp
    iOSApp -->|Local Storage| SwiftData[(SwiftData<br/>Local Database)]
```

### Data Flow: Recipe Extraction

```mermaid
sequenceDiagram
    participant User
    participant ShareExt as Share Extension
    participant iOSApp as iOS App
    participant Backend as Backend API
    participant OpenAI as OpenAI API
    participant SwiftData as Local Storage
    
    User->>ShareExt: Share video URL from social media
    ShareExt->>iOSApp: Open app with video URL
    iOSApp->>Backend: POST /extract_recipe {url}
    Backend->>OpenAI: Transcribe audio (Whisper)
    OpenAI-->>Backend: Transcript
    Backend->>OpenAI: Extract recipe (GPT-4o)
    OpenAI-->>Backend: Structured recipe
    Backend-->>iOSApp: Recipe JSON response
    iOSApp->>SwiftData: Save recipe locally
    iOSApp-->>User: Display recipe card
```

### Data Flow: Subscription Purchase

```mermaid
sequenceDiagram
    participant User
    participant PaywallView as PaywallView
    participant SubMgr as SubscriptionManager
    participant RCService as RevenueCatService
    participant RevenueCat as RevenueCat API
    participant StoreKit as Apple StoreKit
    
    User->>PaywallView: Tap subscription option
    PaywallView->>SubMgr: purchasePackage(package)
    SubMgr->>RCService: purchase(package)
    RCService->>RevenueCat: Purchases.shared.purchase()
    RevenueCat->>StoreKit: Process purchase
    StoreKit-->>RevenueCat: Transaction receipt
    RevenueCat-->>RCService: CustomerInfo + transaction
    RCService-->>SubMgr: Update isPro status
    SubMgr->>SubMgr: Update @Published properties
    SubMgr-->>PaywallView: Purchase complete
    PaywallView-->>User: Show success, dismiss paywall
```

---

## iOS App Architecture

### Architectural Pattern

The app follows the **MVVM (Model-View-ViewModel)** pattern with SwiftUI:

- **Models**: SwiftData entities and data structures
- **Views**: SwiftUI views that observe state
- **ViewModels**: ObservableObject classes that manage state and business logic

### Component Organization

```
VideotoRecipe/
├── Models/                    # Data models and services
│   ├── Recipe.swift          # Recipe SwiftData model
│   ├── Ingredient.swift       # Ingredient SwiftData model
│   ├── User.swift             # User preferences model
│   ├── MealPlan.swift         # Meal plan model
│   ├── ShoppingList.swift     # Shopping list model
│   ├── Chat.swift             # Chat conversation model
│   ├── RevenueCatService.swift    # RevenueCat SDK wrapper
│   ├── SubscriptionManager.swift  # Subscription state management
│   ├── RecipeAPIClient.swift     # Backend API client
│   └── ...
├── Views/                     # SwiftUI views
│   ├── MainTabView.swift      # Main tab bar container
│   ├── RecipeListView.swift   # Recipe collection view
│   ├── MealPlannerView.swift  # Meal planning interface
│   ├── ShoppingListTabView.swift  # Shopping list view
│   ├── RecipeChatbotView.swift    # AI chatbot interface
│   ├── PaywallView.swift      # Subscription paywall
│   ├── Onboarding/            # Onboarding flow views
│   └── ...
├── Components/                # Reusable UI components
│   ├── RecipeCard.swift       # Recipe card component
│   ├── IngredientRow.swift    # Ingredient display component
│   └── ...
├── RecipeShareExtension/      # Share extension for recipes
├── VideoShareExtension/       # Share extension for videos
└── VideotoRecipeApp.swift     # App entry point
```

### State Management

#### ObservableObject Pattern

The app uses `ObservableObject` and `@Published` properties for reactive state management:

```swift
@MainActor
class SubscriptionManager: NSObject, ObservableObject {
    @Published var isPro: Bool = false
    @Published var customerInfo: CustomerInfo? = nil
    @Published var offerings: Offerings? = nil
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?
    
    // Views automatically update when these properties change
}
```

#### Environment Objects

Key services are injected via `@EnvironmentObject`:

```swift
struct MainTabView: View {
    @EnvironmentObject var subscriptionManager: SubscriptionManager
    
    var body: some View {
        // Access subscriptionManager throughout the view hierarchy
    }
}
```

#### Singleton Pattern

Shared services use the singleton pattern:

```swift
class RecipeAPIClient: ObservableObject {
    static let shared = RecipeAPIClient()
    private init() {}
}
```

### App Initialization

The app initializes in `VideotoRecipeApp.swift`:

```swift
@main
struct VideotoRecipeApp: App {
    @StateObject private var subscriptionManager = SubscriptionManager()
    
    var sharedModelContainer: ModelContainer = {
        let schema = Schema([
            Recipe.self,
            Ingredient.self,
            User.self,
            MealPlan.self,
            ShoppingListItem.self,
            Chat.self,
            ChatMessageModel.self,
        ])
        let modelConfiguration = ModelConfiguration(schema: schema, isStoredInMemoryOnly: false)
        return try ModelContainer(for: schema, configurations: [modelConfiguration])
    }()
    
    var body: some Scene {
        WindowGroup {
            MainTabView()
                .environmentObject(subscriptionManager)
                .modelContainer(sharedModelContainer)
        }
    }
}
```

### Share Extension Integration

The app includes two share extensions for seamless video URL sharing:

1. **RecipeShareExtension**: Handles recipe-related shares
2. **VideoShareExtension**: Handles video URL shares

**App Group Configuration:**
- App Group ID: `group.com.mazen.ScrumpyApp.shared`
- Shared UserDefaults for communication between extensions and main app
- Deep linking via custom URL scheme: `scrumpy://fromShare`

**Share Flow:**
1. User shares video URL from social media app
2. Share extension extracts URL and stores in App Group UserDefaults
3. Share extension opens main app via deep link
4. Main app polls for shared URL and displays `AddRecipeView`

---

## Backend Integration

### API Client

The app uses `RecipeAPIClient` singleton for all backend communication:

**Base URL:** `https://savorbackend.onrender.com`

**Key Methods:**

```swift
class RecipeAPIClient: ObservableObject {
    static let shared = RecipeAPIClient()
    static let baseURL = "https://savorbackend.onrender.com"
    
    // Extract recipe from video URL
    func extractRecipe(from videoURL: String) async throws -> RecipeResponse
    
    // Create recipe via chatbot (SSE streaming)
    func createRecipeFromChat(
        conversation: [ChatMessageRequest],
        message: String?,
        images: [UIImage]?,
        audioData: Data?
    ) async throws -> AsyncThrowingStream<ChatStreamEvent, Error>
    
    // Generate meal plan
    func generateMealPlan(
        userPreferences: UserPreferences,
        startDate: Date,
        endDate: Date,
        recipes: [Recipe]
    ) async throws -> MealPlanResponse
    
    // Create Instacart shopping list
    func createInstacartList(recipe: Recipe) async throws -> InstacartResponse
}
```

### API Endpoints

#### Extract Recipe from Video
- **Endpoint**: `POST /extract_recipe`
- **Request**: `{ "url": "https://instagram.com/watch?v=..." }`
- **Response**: Complete recipe with ingredients, steps, nutrition, creator info

#### Create Recipe via Chat
- **Endpoint**: `POST /chat/create_recipe`
- **Request**: Conversation history, message, images (base64), audio (base64)
- **Response**: Server-Sent Events (SSE) stream with instant messages and recipe

#### Generate Meal Plan
- **Endpoint**: `POST /generate_meal_plan`
- **Request**: User preferences, date range, available recipes
- **Response**: Meal plan with recipes assigned to dates/meals

#### Create Instacart List
- **Endpoint**: `POST /create_instacart_list`
- **Request**: Recipe data
- **Response**: Instacart shopping list URL

### Server-Sent Events (SSE)

The chatbot uses SSE for real-time streaming responses:

```swift
let (asyncBytes, response) = try await URLSession.shared.bytes(for: request)

for try await line in asyncBytes.lines {
    if line.hasPrefix("data: ") {
        let jsonString = String(line.dropFirst(6))
        let event = try JSONDecoder().decode(ChatStreamEvent.self, from: jsonString.data(using: .utf8)!)
        // Process event (instant_message, recipe, done)
    }
}
```

### Error Handling

The API client implements comprehensive error handling:

```swift
enum APIError: Error {
    case invalidURL
    case networkError(Error)
    case decodingError(Error)
    case serverError(String)
    case timeout
    case invalidImage
}
```

All API calls include:
- Timeout handling (120 seconds for video processing)
- Retry logic for transient failures
- User-friendly error messages
- Loading state management

---

## Backend Architecture

### Backend System Architecture

```mermaid
graph TB
    Client[Client Application]
    NextJS[Next.js API Routes]
    FastAPI[FastAPI Backend]
    DB[(PostgreSQL Database)]
    
    OpenAI[OpenAI API]
    Instacart[Instacart API]
    Serper[Serper.dev API]
    Gemini[Gemini API]
    
    Client -->|HTTP Requests| NextJS
    Client -->|Direct API Calls| FastAPI
    NextJS -->|Proxy/Enhance| FastAPI
    
    FastAPI -->|Cache Lookup/Store| DB
    FastAPI -->|Transcribe| OpenAI
    FastAPI -->|Extract Recipe| OpenAI
    FastAPI -->|OCR Fallback| Gemini
    FastAPI -->|Shopping Lists| Instacart

    FastAPI -->|Ingredient Images| Serper
```

### Backend Data Flow: Recipe Extraction

```mermaid
sequenceDiagram
    participant Client
    participant API as FastAPI/Next.js
    participant Cache as PostgreSQL Cache
    participant Video as Video Platform
    participant OpenAI as OpenAI API
    participant USDA as USDA API
    participant Instacart as Instacart API
    
    Client->>API: POST /extract_recipe {url}
    API->>Cache: Check cached recipe
    alt Cache Hit
        Cache-->>API: Return cached recipe
        API-->>Client: Recipe (cached)
    else Cache Miss
        API->>Video: Download audio/metadata
        Video-->>API: Audio file + metadata
        API->>OpenAI: Transcribe audio (Whisper)
        OpenAI-->>API: Transcript
        API->>OpenAI: Extract recipe (GPT-4o)
        OpenAI-->>API: Structured recipe
        API->>USDA: Get nutrition data
        USDA-->>API: Nutrition info
        API->>Instacart: Create shopping list
        Instacart-->>API: Shopping URL
        API->>Cache: Store recipe
        API-->>Client: Complete recipe
    end
```

### Caching Strategy

The backend implements intelligent URL normalization and caching:

1. **URL Normalization**: Removes tracking parameters, normalizes platform-specific URLs
2. **Cache Lookup**: Checks normalized URL before processing
3. **Cache Storage**: Stores full recipe data with extraction method and metadata
4. **Cache Statistics**: Tracks access counts and hit rates

### Progress Tracking

- In-memory progress store for extraction operations
- Real-time progress updates via polling endpoint
- Progress stages: downloading → transcribing → extracting → calculating_nutrition → complete

---

## Backend API Endpoints

### FastAPI Backend Endpoints

Base URL: `https://savorbackend.onrender.com` (production) or `http://localhost:8000` (development)

#### Health Check

**GET** `/`

Returns API status.

**Response:**
```json
{
  "message": "Video to Recipe API",
  "status": "running"
}
```

#### Get Extraction Progress

**GET** `/progress/{extraction_id}`

Get real-time progress of a recipe extraction.

**Parameters:**
- `extraction_id` (path): Unique extraction identifier

**Response:**
```json
{
  "step": "extracting",
  "current_ingredient": "butter",
  "progress": 0.65,
  "message": "Finding nutrition for butter..."
}
```

**Progress Steps:**
- `downloading`: Downloading video/audio
- `transcribing`: Converting audio to text
- `extracting`: Extracting recipe structure
- `calculating_nutrition`: Fetching nutrition data
- `complete`: Extraction finished

#### Cache Statistics

**GET** `/cache/stats`

Get statistics about the recipe cache.

**Response:**
```json
{
  "enabled": true,
  "connected": true,
  "total_cached_recipes": 150,
  "total_cache_hits": 450,
  "most_accessed": {
    "title": "Chocolate Chip Cookies",
    "access_count": 25
  },
  "most_recent": {
    "title": "Pasta Carbonara",
    "cached_at": "2024-01-15T10:30:00Z"
  }
}
```

#### Extract Recipe from Video

**POST** `/extract_recipe`

Extract a structured recipe from a video URL.

**Request Body:**
```json
{
  "url": "https://www.youtube.com/watch?v=VIDEO_ID"
}
```

**Response:**
```json
{
  "extraction_id": "uuid-here",
  "title": "Hollandaise Sauce",
  "ingredients": [
    {
      "name": "egg yolks",
      "quantity": "3",
      "unit": null,
      "grams": 60.0,
      "amazon_url": "https://www.amazon.com/s?k=egg+yolks",
      "amazon_product": {
        "type": "search",
        "search_url": "https://www.amazon.com/s?k=egg+yolks",
        "product_name": "egg yolks",
        "affiliate_link": "https://www.amazon.com/s?k=egg+yolks&tag=your-tag"
      },
      "instacart_url": null,
      "image_url": "https://example.com/egg-yolks.jpg",
      "nutrition": {
        "calories": 180.0,
        "protein": 9.0,
        "carbs": 1.8,
        "fat": 15.0
      }
    }
  ],
  "steps": [
    "Separate egg yolks from whites",
    "Melt butter until hot",
    "Blend egg yolks while slowly adding hot butter"
  ],
  "transcript": "So to make hollandaise sauce...",
  "saved_video_url": "/videos/video_id.mp4",
  "extraction_method": "audio_transcription",
  "thumbnail_url": "https://i.ytimg.com/vi/VIDEO_ID/hqdefault.jpg",
  "servings": 4,
  "nutrition_per_serving": {
    "calories": 250.0,
    "protein": 5.0,
    "carbs": 2.0,
    "fat": 24.0
  },
  "instacart_shopping_url": null,
  "creator": {
    "name": "Chef Name",
    "channel_name": "Cooking Channel",
    "profile_url": "https://youtube.com/@channel",
    "profile_image_url": "https://example.com/profile.jpg",
    "platform": "youtube"
  },
  "cached": false
}
```

**Supported Platforms:**
- YouTube
- TikTok
- Instagram
- Other platforms supported by yt-dlp

**Extraction Methods:**
1. Video description (fastest)
2. Captions/subtitles
3. Audio transcription (Whisper)
4. OCR fallback (Gemini Vision)

#### Create Recipe via Chatbot

**POST** `/chat/create_recipe`

Create or update a recipe through conversational interface. Uses Server-Sent Events (SSE) for streaming responses.

**Request Body:**
```json
{
  "conversation": [
    {
      "role": "user",
      "content": "I want to make pasta carbonara"
    },
    {
      "role": "assistant",
      "content": "Great! I can help you make pasta carbonara. What ingredients do you have?",
      "recipe_context": null
    }
  ],
  "message": "I have eggs, pasta, bacon, and parmesan",
  "images": null,
  "audio_base64": null
}
```

**Response (SSE Stream):**
```
event: instant_message
data: {"message": "Perfect! Let me create a pasta carbonara recipe for you..."}

event: recipe
data: {
  "mode": "recipe",
  "assistant_message": "Here's your pasta carbonara recipe!",
  "recipe": {
    "title": "Pasta Carbonara",
    "ingredients": [...],
    "steps": [...]
  },
  "needs_clarification": false
}

event: done
data: {}
```

**Features:**
- Multi-turn conversation support
- Recipe context preservation
- Voice input (base64 audio)
- Image input (base64 images)
- Recipe updates from conversation

#### Create Instacart Shopping List

**POST** `/create_instacart_list`

Create an Instacart recipe page for a recipe.

**Request Body:**
```json
{
  "title": "Hollandaise Sauce",
  "ingredients": [
    {
      "name": "egg yolks",
      "quantity": "3",
      "unit": null
    },
    {
      "name": "butter",
      "quantity": "1",
      "unit": "cup"
    }
  ],
  "steps": ["Step 1", "Step 2"],
  "transcript": "...",
  "thumbnail_url": "https://example.com/thumb.jpg",
  "servings": 4
}
```

**Response:**
```json
{
  "instacart_url": "https://www.instacart.com/products/recipe/abc123",
  "success": true
}
```

#### Get Ingredient Images

**POST** `/get_ingredient_images`

Get product images for multiple ingredients.

**Request Body:**
```json
{
  "ingredients": ["butter", "eggs", "flour"],
  "size": "medium"
}
```

**Response:**
```json
{
  "ingredients": [
    {
      "name": "butter",
      "image_url": "https://example.com/butter.jpg",
      "image_source": "serper"
    },
    {
      "name": "eggs",
      "image_url": "https://example.com/eggs.jpg",
      "image_source": "kroger"
    },
    {
      "name": "flour",
      "image_url": null,
      "image_source": "emoji"
    }
  ],
  "found_count": 2,
  "total_count": 3
}
```

**Image Sources:**
- `serper`: Serper.dev Google Images
- `kroger`: Kroger Product API
- `emoji`: Emoji fallback

#### Get Single Ingredient Image

**GET** `/ingredient_image/{ingredient_name}`

Get a single ingredient image.

**Parameters:**
- `ingredient_name` (path): Name of the ingredient

**Response:**
```json
{
  "ingredient_name": "butter",
  "image_url": "https://example.com/butter.jpg",
  "image_source": "serper"
}
```

#### Generate Meal Plan

**POST** `/generate_meal_plan`

Generate a meal plan using AI based on user preferences and available recipes.

**Request Body:**
```json
{
  "date_range": {
    "start_date": "2024-01-15",
    "end_date": "2024-01-21"
  },
  "preferences": {
    "dietary_restrictions": ["vegetarian"],
    "cuisine_preferences": ["italian", "mexican"],
    "avoid_ingredients": ["nuts"],
    "preferred_meals": ["breakfast", "dinner"]
  },
  "available_recipes": [
    {
      "id": "recipe-1",
      "title": "Pasta Carbonara",
      "ingredients": [...],
      "steps": [...]
    }
  ]
}
```

**Response:**
```json
{
  "meal_plan": {
    "start_date": "2024-01-15",
    "end_date": "2024-01-21",
    "meals": [
      {
        "date": "2024-01-15",
        "meal_type": "breakfast",
        "recipe_id": "recipe-1",
        "recipe_title": "Pasta Carbonara"
      }
    ]
  },
  "shopping_lists": {
    "instacart_url": "https://www.instacart.com/products/recipe/abc123"
  }
}
```

### Next.js API Routes

Base URL: `/api` (relative to Next.js app)

#### Extract Recipe (Next.js)

**POST** `/api/extract-recipe`

Wrapper around FastAPI endpoint with additional processing.

**Request/Response:** Same as FastAPI `/extract_recipe`

#### Create Instacart List (Next.js)

**POST** `/api/create-instacart-list`

**Request/Response:** Same as FastAPI `/create_instacart_list`

#### Search Ingredient

**POST** `/api/search-ingredient?ingredient={name}`

Search for an ingredient on Amazon.

**Query Parameters:**
- `ingredient`: Ingredient name to search

**Response:**
```json
{
  "type": "search",
  "search_url": "https://www.amazon.com/s?k=butter",
  "product_name": "butter",
  "affiliate_link": "https://www.amazon.com/s?k=butter&tag=your-tag"
}
```

#### Get Ingredient Images (Next.js)

**POST** `/api/get-ingredient-images`

**Request/Response:** Same as FastAPI `/get_ingredient_images`

#### Get Single Ingredient Image (Next.js)

**GET** `/api/ingredient-image/[name]`

**Request/Response:** Same as FastAPI `/ingredient_image/{ingredient_name}`

#### Get Progress (Next.js)

**GET** `/api/progress/[id]`

**Request/Response:** Same as FastAPI `/progress/{extraction_id}`

---

## Backend External Integrations

### OpenAI Integration

**Services Used:**
- **Whisper-1**: Audio transcription
- **GPT-4o**: Recipe extraction and structuring
- **GPT-4 Vision**: OCR fallback for video frames

**Configuration:**
- API Key: `OPENAI_API_KEY`
- Model: `gpt-4o` (configurable via `GPT_MODEL`)
- Whisper Model: `whisper-1` (configurable via `WHISPER_MODEL`)

**Cost Estimates:**
- Whisper: $0.006 per minute of audio
- GPT-4o: ~$0.01-0.03 per recipe extraction
- Total per recipe: ~$0.03-0.04 (for 3-minute video)

**Rate Limits:**
- Default: No rate limiting (handled by OpenAI)
- Configurable via `ENABLE_RATE_LIMITING` and `RATE_LIMIT_PER_MINUTE`

### Instacart Developer Platform

**Endpoints Used:**
- `/products/recipe`: Create recipe shopping pages
- `/products/availability`: Check product availability
- `/retailers`: Get available retailers

**Configuration:**
- API Key: `INSTACART_API_KEY`
- Environment: Development (`INSTACART_USE_PRODUCTION=false`) or Production
- Base URLs:
  - Dev: `https://connect.dev.instacart.tools/idp/v1`
  - Prod: `https://connect.instacart.com/idp/v1`

**Features:**
- Recipe page creation with metadata
- Store availability checking
- Product search and matching

### Kroger Product API

**Purpose:** Fetch product images for ingredients

**Configuration:**
- Client ID: `KROGER_CLIENT_ID`
- Client Secret: `KROGER_CLIENT_SECRET`
- OAuth2 authentication with `product.compact` scope

**Endpoints:**
- `/v1/connect/oauth2/token`: OAuth2 token
- `/v1/products`: Product search

**Image Sizes:** thumbnail, small, medium, large, xlarge

### Amazon Product Advertising API

**Purpose:** Product search and affiliate links

**Configuration:**
- Access Key: `AMAZON_ACCESS_KEY`
- Secret Key: `AMAZON_SECRET_KEY`
- Associate Tag: `AMAZON_ASSOCIATE_TAG`
- Region: `AMAZON_REGION` (default: "US")

**Features:**
- Product search URLs (free)
- Optional: Full PA API integration for product details
- Affiliate link generation

### Serper.dev

**Purpose:** Google Images search for ingredient photos

**Configuration:**
- API Key: `SERPER_API_KEY`

**Features:**
- Fast synchronous image search
- High-quality product images
- Fallback to emoji if no image found

### USDA FoodData Central

**Purpose:** Nutrition data lookup

**Configuration:**
- API Key: `USDA_API_KEY`

**Endpoints:**
- `/fdc/v1/foods/search`: Search foods
- `/fdc/v1/food/{fdcId}`: Get food details

**Features:**
- Comprehensive nutrition database
- AI-assisted food matching
- Per-ingredient and per-serving nutrition calculation

### Google Gemini Vision API

**Purpose:** OCR fallback for recipe text extraction

**Configuration:**
- API Key: `GEMINI_API_KEY`

**Usage:**
- Extracts text from video frames when audio transcription fails
- Processes multiple frames for better accuracy
- Fallback method in extraction pipeline

---

## Backend Database Schema

### cached_recipes Table

PostgreSQL table for storing cached recipes.

**Schema:**
```sql
CREATE TABLE cached_recipes (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    normalized_url TEXT UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    recipe_data JSONB NOT NULL,
    extraction_method TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    access_count INTEGER DEFAULT 1,
    last_accessed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_cached_recipes_url ON cached_recipes (normalized_url);
```

**Fields:**
- `id`: Auto-incrementing primary key
- `normalized_url`: Normalized video URL (unique, indexed)
- `original_url`: Original video URL
- `recipe_data`: Full recipe JSON (JSONB for querying)
- `extraction_method`: How recipe was extracted
- `created_at`: When recipe was cached
- `access_count`: Number of cache hits
- `last_accessed_at`: Last access timestamp

**URL Normalization:**
- Removes tracking parameters (utm_*, fbclid, etc.)
- Normalizes platform-specific URLs (YouTube, TikTok, Instagram)
- Lowercases scheme and netloc
- Removes trailing slashes

---

## Backend Configuration

### Required Environment Variables

#### OpenAI (Required)
```bash
OPENAI_API_KEY=sk-...
```

#### Database (Optional, but recommended)
```bash
DATABASE_URL=postgresql://user:password@host:port/database
CACHE_ENABLED=true
```

### Optional Environment Variables

#### API Configuration
```bash
API_HOST=0.0.0.0
API_PORT=8000
DEBUG=false
```

#### OpenAI Models
```bash
WHISPER_MODEL=whisper-1
GPT_MODEL=gpt-4o
```

#### Instacart
```bash
INSTACART_API_KEY=your_instacart_api_key
INSTACART_USE_PRODUCTION=false  # Set to "true" for production
```

#### Serper.dev
```bash
SERPER_API_KEY=your_serper_api_key
```

#### Gemini (OCR Fallback)
```bash
GEMINI_API_KEY=your_gemini_api_key
```

#### Video Processing
```bash
TEMP_DIR=/tmp  # Optional: custom temp directory
MAX_VIDEO_DURATION=600  # 10 minutes
AUDIO_QUALITY=0  # 0 is best
INSTAGRAM_COOKIES_FILE=/path/to/cookies.txt  # Optional
```

#### Recipe Extraction
```bash
MAX_INGREDIENTS=50
MAX_STEPS=30
```

#### CORS
```bash
ALLOWED_ORIGINS=*  # Comma-separated list
```

#### Rate Limiting
```bash
ENABLE_RATE_LIMITING=false
RATE_LIMIT_PER_MINUTE=10
```

#### Timeouts (seconds)
```bash
TIMEOUT_VIDEO_METADATA=30
TIMEOUT_AUDIO_DOWNLOAD=120
TIMEOUT_TRANSCRIPTION=60
TIMEOUT_RECIPE_EXTRACTION=90
TIMEOUT_OCR_FALLBACK=120
```

#### Logging
```bash
LOG_LEVEL=INFO
LOG_FILE=app.log
```

### Configuration File

Configuration is managed in `backend/config.py` with validation. Run:

```bash
python backend/config.py
```

To validate your configuration.

---

## Backend Error Handling

### Common Error Scenarios

#### 1. Invalid Video URL

**Status:** `400 Bad Request`

**Response:**
```json
{
  "detail": "Invalid video URL or unsupported platform"
}
```

#### 2. Video Not Found

**Status:** `404 Not Found`

**Response:**
```json
{
  "detail": "Video not found or unavailable"
}
```

#### 3. Transcription Failed

**Status:** `500 Internal Server Error`

**Response:**
```json
{
  "detail": "Failed to transcribe audio. Please try again or use a different video."
}
```

#### 4. Recipe Extraction Failed

**Status:** `500 Internal Server Error`

**Response:**
```json
{
  "detail": "Failed to extract recipe from video. The video may not contain recipe information."
}
```

#### 5. API Key Missing

**Status:** `500 Internal Server Error`

**Response:**
```json
{
  "detail": "OpenAI API key not configured"
}
```

#### 6. Database Connection Failed

**Status:** `500 Internal Server Error`

**Response:**
```json
{
  "detail": "Database connection failed. Caching disabled."
}
```

### Error Response Format

All errors follow this format:

```json
{
  "detail": "Error message here"
}
```

For validation errors:

```json
{
  "detail": [
    {
      "loc": ["body", "url"],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
```

### Retry Strategies

1. **Transient Errors**: Retry with exponential backoff
2. **Rate Limiting**: Wait and retry after rate limit window
3. **Network Errors**: Retry up to 3 times
4. **API Errors**: Log and return user-friendly error

### Timeout Handling

All operations have configurable timeouts:
- Video metadata: 30s
- Audio download: 120s
- Transcription: 60s
- Recipe extraction: 90s
- OCR fallback: 120s

Timeout errors return `504 Gateway Timeout`.

---

## Backend Deployment

### Development Setup

1. **Install Dependencies:**
```bash
pip install -r requirements.txt
```

2. **Configure Environment:**
```bash
cp .env.example .env
# Edit .env with your API keys
```

3. **Start Server:**
```bash
uvicorn backend.app:app --reload --host 0.0.0.0 --port 8000
```

### Production Deployment

#### Render.com (Recommended)

1. **Connect Repository** to Render
2. **Create Web Service** using `render.yaml`
3. **Set Environment Variables** in Render dashboard
4. **Deploy**

The `render.yaml` file includes:
- Build command: `./build.sh`
- Start command: `./start.sh`
- Health check: `GET /`

#### Docker Deployment

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "backend.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Environment Variables in Production

Set all required environment variables in your deployment platform:
- `OPENAI_API_KEY` (required)
- `DATABASE_URL` (recommended)
- `SERPER_API_KEY` (for ingredient images)
- Other API keys as needed

### Database Setup

1. **Create PostgreSQL Database** (Supabase recommended)
2. **Set `DATABASE_URL`** environment variable
3. **Tables are created automatically** on first connection

### Monitoring

- **Health Check**: `GET /`
- **Cache Stats**: `GET /cache/stats`
- **Logs**: Check application logs for errors
- **Progress Tracking**: Monitor extraction progress endpoints

### Scaling Considerations

- **Database Connection Pooling**: Configured in `database.py`
- **Async Operations**: All I/O operations are async
- **Caching**: Reduces API calls and processing time
- **Rate Limiting**: Configure if needed

### Code Structure

```
SavorBackend/
├── backend/
│   ├── app.py              # Main FastAPI application
│   ├── config.py           # Configuration management
│   ├── database.py         # Database operations
│   ├── instacart_api.py    # Instacart integration
│   ├── kroger_api.py       # Kroger integration
│   ├── amazon_api.py       # Amazon integration
│   ├── ocr_helper.py       # OCR fallback
│   └── ingredient_images.py # Image fetching
├── savor-app/
│   └── src/
│       ├── app/api/        # Next.js API routes
│       └── lib/            # Shared libraries
└── requirements.txt        # Python dependencies
```

### Testing

Test the API using:
```bash
python test_api.py
```

Or use the example client:
```bash
python example_client.py
```

### API Documentation

- **FastAPI Docs**: `http://localhost:8000/docs` (Swagger UI)
- **ReDoc**: `http://localhost:8000/redoc`
- **Production**: `https://savorbackend.onrender.com/docs`

---

## RevenueCat Implementation

### Overview

SavorApp uses RevenueCat for subscription management, providing a unified interface for in-app purchases across iOS platforms. The implementation supports free and pro subscription tiers with monthly, yearly, and lifetime options.

### Architecture

```mermaid
graph TB
    PaywallView[PaywallView<br/>SwiftUI]
    SubMgr[SubscriptionManager<br/>ObservableObject]
    RCService[RevenueCatService<br/>SDK Wrapper]
    RevenueCatSDK[RevenueCat SDK<br/>Purchases]
    StoreKit[Apple StoreKit]
    
    PaywallView -->|Observes| SubMgr
    SubMgr -->|Uses| RCService
    RCService -->|Calls| RevenueCatSDK
    RevenueCatSDK -->|Processes| StoreKit
    StoreKit -->|Receipts| RevenueCatSDK
    RevenueCatSDK -->|Updates| RCService
    RCService -->|Notifies| SubMgr
    SubMgr -->|Publishes| PaywallView
```

### Key Components

#### 1. RevenueCatService

**Location**: `VideotoRecipe/Models/RevenueCatService.swift`

Core SDK wrapper that handles all RevenueCat operations:

```swift
@MainActor
class RevenueCatService: ObservableObject {
    static let shared = RevenueCatService()
    
    private var apiKey: String {
        #if DEBUG
        return "test_xOInCpCECcxDjkVgmrzYRXPlqDv"  // Sandbox key
        #else
        return "appl_********"  // Production key
        #endif
    }
    
    private let entitlementID = "Savor Pro"
    
    // Configuration
    func configure()
    
    // Entitlement checking
    func checkEntitlementStatus() async throws -> Bool
    func getCustomerInfo() async throws -> CustomerInfo
    
    // Offerings
    func fetchOfferings() async throws -> Offerings
    
    // Purchases
    func purchase(package: Package) async throws -> (CustomerInfo, Bool)
    func restorePurchases() async throws -> CustomerInfo
    
    // User sync
    func syncUserID(_ userID: String) async throws
}
```

**Key Features:**
- Automatic API key selection based on build configuration
- Single global configuration point
- Comprehensive error handling with custom `RevenueCatError` enum
- Detailed logging for debugging

#### 2. SubscriptionManager

**Location**: `VideotoRecipe/Models/SubscriptionManager.swift`

ObservableObject that manages subscription state for UI binding:

```swift
@MainActor
class SubscriptionManager: NSObject, ObservableObject {
    @Published var isPro: Bool = false
    @Published var customerInfo: CustomerInfo? = nil
    @Published var offerings: Offerings? = nil
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?
    
    private let revenueCatService = RevenueCatService.shared
    
    // Subscription status
    func checkSubscriptionStatus() async
    
    // Offerings
    func loadOfferings(maxRetries: Int = 2) async
    
    // Purchases
    func purchasePackage(_ package: Package) async -> Bool
    func restorePurchases() async
    
    // Helper properties
    var currentSubscriptionPeriod: String?
    var expirationDate: Date?
    var isInTrial: Bool
}
```

**Key Features:**
- Automatic RevenueCat configuration on initialization
- Customer info listener via `PurchasesDelegate`
- Retry logic for offerings fetch (up to 2 retries with exponential backoff)
- User-friendly error messages
- Subscription period detection (Monthly, Yearly, Lifetime, Trial)

#### 3. PaywallView

**Location**: `VideotoRecipe/Views/PaywallView.swift`

SwiftUI view that displays the RevenueCat paywall:

```swift
struct PaywallView: View {
    @EnvironmentObject var subscriptionManager: SubscriptionManager
    @Environment(\.dismiss) var dismiss
    
    var body: some View {
        if subscriptionManager.offerings?.current != nil {
            PaywallViewWrapper(
                offering: subscriptionManager.offerings?.current,
                onPurchaseComplete: { handlePurchaseCompleted() },
                onDismiss: { dismiss() }
            )
        } else if subscriptionManager.isLoading {
            ProgressView()
        } else {
            // Error state with retry
        }
    }
}
```

**Key Features:**
- Uses RevenueCat's `PaywallViewController` (UIKit) wrapped in SwiftUI
- Automatically loads offerings on appear
- Handles purchase completion and dismissal
- Shows loading and error states
- Integrates with custom paywall template from RevenueCat dashboard

### Configuration

#### API Keys

The app uses different API keys for Debug and Release builds:

- **Debug (Sandbox)**: `test_xOInCpCECcxDjkVgmrzYRXPlqDv`
- **Release (Production)**: `appl_******`

API keys are stored directly in `RevenueCatService.swift` and selected at compile time based on build configuration.

#### Entitlement

- **Entitlement ID**: `"Savor Pro"`
- All subscription products (monthly, yearly, lifetime) grant the same entitlement
- App checks `customerInfo.entitlements["Savor Pro"]?.isActive` to determine pro status

#### Initialization

RevenueCat is configured automatically when `SubscriptionManager` is initialized:

```swift
override init() {
    super.init()
    revenueCatService.configure()  // Single global configuration
    setupCustomerInfoListener()
    Task {
        await checkSubscriptionStatus()
        await loadOfferings()
    }
}
```

### Subscription Tiers

#### Free Tier
- Limited chat conversations (10 per day)
- 5 recipe extractions from videos per day

#### Pro Tier (Savor Pro)
- Unlimited chat conversations
- Unlimited recipe extraction features


### Pricing

- **Monthly**: $4.99/month
- **Weekly**: $1.99/month


Prices are configured in App Store Connect and displayed dynamically via `package.storeProduct.localizedPriceString`.

### Purchase Flow

1. **User taps subscription option** in `PaywallView`
2. **PaywallView** calls `subscriptionManager.purchasePackage(package)`
3. **SubscriptionManager** calls `revenueCatService.purchase(package)`
4. **RevenueCatService** calls `Purchases.shared.purchase(package: package)`
5. **RevenueCat SDK** presents StoreKit purchase sheet
6. **User completes purchase** via Apple's native UI
7. **StoreKit** processes payment and returns receipt
8. **RevenueCat** validates receipt and updates customer info
9. **RevenueCatService** receives updated `CustomerInfo`
10. **SubscriptionManager** updates `@Published var isPro = true`
11. **PaywallView** automatically dismisses (observes `isPro` change)

### Restore Purchases

Users can restore previous purchases:

```swift
func restorePurchases() async {
    let customerInfo = try await revenueCatService.restorePurchases()
    self.customerInfo = customerInfo
    self.isPro = customerInfo.entitlements["Savor Pro"]?.isActive == true
}
```

### Customer Info Updates

The app listens for real-time customer info updates via `PurchasesDelegate`:

```swift
extension SubscriptionManager: PurchasesDelegate {
    nonisolated func purchases(_ purchases: Purchases, receivedUpdated customerInfo: CustomerInfo) {
        Task { @MainActor in
            self.customerInfo = customerInfo
            self.isPro = customerInfo.entitlements["Savor Pro"]?.isActive == true
        }
    }
}
```

This ensures the app stays in sync with subscription status changes (renewals, cancellations, etc.).

### Cross-Device Sync

The app supports syncing subscription status across devices:

```swift
func syncUserID(_ userID: String) async throws {
    let (customerInfo, created) = try await Purchases.shared.logIn(userID)
    // Subscription status now synced across devices
}
```

### Error Handling

Custom error enum for user-friendly messages:

```swift
enum RevenueCatError: LocalizedError {
    case notConfigured
    case noOfferingsFound
    case purchaseCancelled
    case purchaseFailed(String)
    case restoreFailed(String)
}
```

All errors are caught and converted to user-friendly messages in `SubscriptionManager`.

### Testing

#### Sandbox Testing
- Use Debug build configuration (automatically uses sandbox API key)
- Test with sandbox Apple ID in Simulator or TestFlight
- Products must be approved in App Store Connect for TestFlight

#### Production Testing
- Use Release build configuration (uses production API key)
- Requires TestFlight or App Store distribution
- Products must be approved and available in App Store Connect

### Troubleshooting

Common issues and solutions:

1. **No offerings found**
   - Verify offerings are configured in RevenueCat dashboard
   - Ensure products are linked to offerings
   - Check API key matches environment (Debug vs Release)
   - For TestFlight: Products must be approved (not just "Ready to Submit")

2. **Purchase fails**
   - Verify StoreKit configuration in App Store Connect
   - Check product IDs match between App Store Connect and RevenueCat
   - Ensure user is signed in with valid Apple ID
   - Check network connectivity

3. **Subscription status not updating**
   - Verify `PurchasesDelegate` is set correctly
   - Check entitlement ID matches RevenueCat dashboard
   - Ensure customer info is refreshed after purchase

---

## Data Models

### SwiftData Schema

The app uses SwiftData for local persistence with the following models:

```swift
let schema = Schema([
    Recipe.self,
    Ingredient.self,
    User.self,
    MealPlan.self,
    ShoppingListItem.self,
    Chat.self,
    ChatMessageModel.self,
])
```

### Recipe Model

**Location**: `VideotoRecipe/Models/Recipe.swift`

```swift
@Model
final class Recipe {
    var id: UUID
    var title: String
    var ingredients: [Ingredient]
    var steps: [String]
    var transcript: String
    var videoURL: String
    var savedVideoURL: String?
    var thumbnailURL: String?
    var servings: Int?
    var nutritionPerServing: NutritionInfoData?
    var instacartShoppingURL: String?
    var creator: CreatorInfoData?
    var createdAt: Date
}
```

**Relationships:**
- `ingredients`: One-to-many relationship with `Ingredient`
- Referenced by `MealPlan` (optional relationship)

### Ingredient Model

**Location**: `VideotoRecipe/Models/Ingredient.swift`

```swift
@Model
final class Ingredient {
    var id: UUID
    var name: String
    var quantity: String?
    var unit: String?
    var grams: Double?
    var amazonUrl: String?
    var instacartUrl: String?
    var imageURL: String?
}
```

**Relationships:**
- Belongs to `Recipe` (many-to-one)

### User Model

**Location**: `VideotoRecipe/Models/User.swift`

```swift
@Model
final class User {
    var id: String  // UUID string
    var email: String?
    var name: String?
    var dietaryRestrictions: [String] = []
    var avoidIngredients: [String] = []
    var weeklyGroceryBudget: Double = 150.0
    var estimatedSavings: Double = 0.0
    var householdSize: Int = 2
    var cookingSkillLevel: String?
    var cookingVarietyPreference: String?
    var hasCompletedOnboarding: Bool = false
    var createdAt: Date
    var lastLoginAt: Date
}
```

### MealPlan Model

**Location**: `VideotoRecipe/Models/MealPlan.swift`

```swift
@Model
final class MealPlan {
    var id: UUID
    var date: Date
    var mealType: String  // "Breakfast", "Lunch", "Dinner"
    var recipe: Recipe?  // Optional relationship
    var includedIngredients: [String]  // Array of ingredient IDs
}
```

**Relationships:**
- `recipe`: Optional many-to-one relationship with `Recipe`

### ShoppingListItem Model

**Location**: `VideotoRecipe/Models/ShoppingList.swift`

```swift
@Model
final class ShoppingListItem {
    var id: UUID
    var ingredientName: String
    var quantity: String?
    var unit: String?
    var price: Double
    var isChecked: Bool
    var fromRecipeTitle: String?
    var category: String
    var addedDate: Date
}
```

### Chat Model

**Location**: `VideotoRecipe/Models/Chat.swift`

```swift
@Model
final class Chat {
    var id: UUID
    var title: String
    var createdAt: Date
    var updatedAt: Date
    @Relationship(deleteRule: .cascade) var messages: [ChatMessageModel]
    var conversationHistoryData: Data?  // JSON for API continuity
}
```

**Relationships:**
- `messages`: One-to-many with cascade delete

### ChatMessageModel

**Location**: `VideotoRecipe/Models/Chat.swift`

```swift
@Model
final class ChatMessageModel {
    var id: UUID
    var text: String
    var role: String  // "user" or "assistant"
    var createdAt: Date
    // Belongs to Chat
}
```

### Data Access Patterns

#### Querying Data

```swift
@Query private var recipes: [Recipe]
@Query private var users: [User]
@Query private var mealPlans: [MealPlan]
```

#### Saving Data

```swift
modelContext.insert(recipe)
try modelContext.save()
```

#### Filtering

```swift
let descriptor = FetchDescriptor<Recipe>(
    predicate: #Predicate<Recipe> { recipe in
        recipe.title.contains(searchText)
    }
)
let recipes = try modelContext.fetch(descriptor)
```

---

## Development Setup

### Prerequisites

- **Xcode 15+**: Required for Swift 5.9+ and SwiftData support
- **iOS 17+**: Minimum deployment target
- **Apple Developer Account**: For App Store Connect and provisioning
- **RevenueCat Account**: For subscription management

### Project Structure

```
SavorApp/
├── VideotoRecipe/              # Main app target
│   ├── Models/                 # Data models and services
│   ├── Views/                  # SwiftUI views
│   ├── Components/             # Reusable UI components
│   ├── Assets.xcassets/        # Images and assets
│   └── VideotoRecipeApp.swift  # App entry point
├── RecipeShareExtension/       # Share extension target
├── VideoShareExtension/        # Share extension target
├── VideotoRecipe.xcodeproj/    # Xcode project
└── Documentation/              # Documentation files
```

### Configuration

#### App Group Setup

1. **Create App Group** in Apple Developer Portal:
   - Identifier: `group.com.mazen.ScrumpyApp.shared`
   - Enable for main app and both share extensions

2. **Configure Entitlements**:
   - Add App Groups capability to all targets
   - Use same group identifier: `group.com.mazen.ScrumpyApp.shared`

#### RevenueCat Setup

1. **Create RevenueCat Project**:
   - Sign up at https://app.revenuecat.com
   - Create new project
   - Add iOS app

2. **Configure API Keys**:
   - Get Test Store API key (for Debug builds)
   - Get Production API key (for Release builds)
   - Update `RevenueCatService.swift` with keys

3. **Configure Products**:
   - Create products in App Store Connect:
     - Monthly subscription (e.g., `com.mazen.savorapp.pro.monthly`)
     - Yearly subscription (e.g., `com.mazen.savorapp.pro.yearly`)
     - Lifetime purchase (e.g., `com.mazen.savorapp.pro.lifetime`)
   - Link products in RevenueCat dashboard
   - Create offering with all products
   - Set offering as "Current"

4. **Configure Paywall**:
   - Design paywall template in RevenueCat dashboard
   - Publish paywall template
   - Link to current offering

#### Backend Configuration

1. **Set Backend URL**:
   - Update `RecipeAPIClient.baseURL` if needed
   - Default: `https://savorbackend.onrender.com`

2. **Environment Variables** (Backend):
   - See [Backend Configuration](#backend-configuration) section for complete list
   - Required: `OPENAI_API_KEY`
   - Optional: Database, Instacart, Kroger, etc.

### Building and Running

1. **Open Project**:
   ```bash
   open VideotoRecipe.xcodeproj
   ```

2. **Select Target**:
   - Main app: `VideotoRecipe`
   - Share extensions: `RecipeShareExtension`, `VideoShareExtension`

3. **Select Scheme**:
   - Choose simulator or connected device

4. **Build and Run**:
   - Press `Cmd+R` or click Run button
   - App will install and launch

### Testing Subscriptions

#### Sandbox Testing

1. **Use Debug Configuration**:
   - Automatically uses sandbox API key
   - Test purchases use sandbox Apple ID

2. **Create Sandbox Tester**:
   - App Store Connect → Users and Access → Sandbox Testers
   - Create test account
   - Sign in on device/simulator

3. **Test Purchase Flow**:
   - Launch app
   - Navigate to paywall
   - Attempt purchase
   - Use sandbox Apple ID when prompted

#### TestFlight Testing

1. **Upload Build**:
   - Archive app in Xcode
   - Upload to App Store Connect
   - Wait for processing

2. **Configure TestFlight**:
   - Add internal/external testers
   - Ensure products are approved (not just "Ready to Submit")

3. **Test**:
   - Install via TestFlight
   - Test purchase flow with real Apple ID
   - Purchases are free in TestFlight

### Debugging

#### RevenueCat Logging

The app includes detailed logging for RevenueCat operations:

```swift
#if DEBUG
Purchases.logLevel = .debug
#else
Purchases.logLevel = .info
#endif
```

Check Xcode console for:
- Configuration status
- Offerings fetch results
- Purchase flow steps
- Error details

#### Common Issues

1. **No offerings found**:
   - Check console logs for API key and environment
   - Verify offerings in RevenueCat dashboard
   - Ensure products are approved in App Store Connect

2. **Purchase fails**:
   - Check StoreKit configuration
   - Verify product IDs match
   - Check network connectivity
   - Review error logs in console

3. **Subscription status not updating**:
   - Verify `PurchasesDelegate` is set
   - Check entitlement ID matches dashboard
   - Refresh customer info manually

---

## Additional Resources

### Documentation Files

- **App Summary**: `APP_SUMMARY.md` - Feature overview and user workflows
- **RevenueCat Pricing Guide**: `REVENUECAT_PRICING_UPDATE_GUIDE.md` - Pricing configuration
- **RevenueCat Troubleshooting**: `REVENUECAT_TESTFLIGHT_TROUBLESHOOTING.md` - Common issues

### External Resources

- **RevenueCat Documentation**: https://docs.revenuecat.com
- **SwiftData Documentation**: https://developer.apple.com/documentation/swiftdata
- **SwiftUI Documentation**: https://developer.apple.com/documentation/swiftui
- **Backend API**: https://savorbackend.onrender.com/docs (FastAPI Swagger UI)

---

**Version**: 1.0.0
