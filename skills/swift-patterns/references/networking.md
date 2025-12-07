# Networking

## URLSession + Async/Await (Recommended)

### Basic GET Request

```swift
func fetchUsers() async throws -> [User] {
    let url = URL(string: "https://api.example.com/users")!
    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw APIError.invalidResponse
    }

    return try JSONDecoder().decode([User].self, from: data)
}
```

### POST Request with Body

```swift
func createUser(_ user: CreateUserRequest) async throws -> User {
    var request = URLRequest(url: URL(string: "https://api.example.com/users")!)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(user)

    let (data, response) = try await URLSession.shared.data(for: request)

    guard let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 201 else {
        throw APIError.invalidResponse
    }

    return try JSONDecoder().decode(User.self, from: data)
}
```

## API Client Pattern

### Complete Implementation

```swift
// APIError.swift
enum APIError: LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(statusCode: Int, message: String?)
    case decodingError(Error)
    case networkError(Error)
    case unauthorized

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "Invalid URL"
        case .invalidResponse:
            return "Invalid response from server"
        case .httpError(let code, let message):
            return message ?? "HTTP Error: \(code)"
        case .decodingError(let error):
            return "Decoding error: \(error.localizedDescription)"
        case .networkError(let error):
            return error.localizedDescription
        case .unauthorized:
            return "Unauthorized"
        }
    }
}

// Endpoint.swift
enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case patch = "PATCH"
    case delete = "DELETE"
}

protocol Endpoint {
    var path: String { get }
    var method: HTTPMethod { get }
    var headers: [String: String]? { get }
    var body: Encodable? { get }
    var queryItems: [URLQueryItem]? { get }
}

extension Endpoint {
    var headers: [String: String]? { nil }
    var body: Encodable? { nil }
    var queryItems: [URLQueryItem]? { nil }
}

// APIClient.swift
actor APIClient {
    static let shared = APIClient()

    private let baseURL: URL
    private let session: URLSession
    private let decoder: JSONDecoder
    private let encoder: JSONEncoder
    private var tokenManager: TokenManager

    init(
        baseURL: URL = URL(string: "https://api.example.com")!,
        session: URLSession = .shared
    ) {
        self.baseURL = baseURL
        self.session = session
        self.decoder = JSONDecoder()
        self.decoder.dateDecodingStrategy = .iso8601
        self.encoder = JSONEncoder()
        self.encoder.dateEncodingStrategy = .iso8601
        self.tokenManager = TokenManager()
    }

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        let request = try buildRequest(for: endpoint)
        return try await performRequest(request)
    }

    func requestVoid(_ endpoint: Endpoint) async throws {
        let request = try buildRequest(for: endpoint)
        let (_, response) = try await session.data(for: request)
        try validateResponse(response)
    }

    private func buildRequest(for endpoint: Endpoint) throws -> URLRequest {
        var components = URLComponents(url: baseURL.appendingPathComponent(endpoint.path), resolvingAgainstBaseURL: true)
        components?.queryItems = endpoint.queryItems

        guard let url = components?.url else {
            throw APIError.invalidURL
        }

        var request = URLRequest(url: url)
        request.httpMethod = endpoint.method.rawValue

        // Default headers
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("application/json", forHTTPHeaderField: "Accept")

        // Auth token
        if let token = tokenManager.accessToken {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        // Custom headers
        endpoint.headers?.forEach { request.setValue($1, forHTTPHeaderField: $0) }

        // Body
        if let body = endpoint.body {
            request.httpBody = try encoder.encode(body)
        }

        return request
    }

    private func performRequest<T: Decodable>(_ request: URLRequest) async throws -> T {
        let (data, response) = try await session.data(for: request)
        try validateResponse(response)

        do {
            return try decoder.decode(T.self, from: data)
        } catch {
            throw APIError.decodingError(error)
        }
    }

    private func validateResponse(_ response: URLResponse) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }

        switch httpResponse.statusCode {
        case 200...299:
            return
        case 401:
            throw APIError.unauthorized
        default:
            throw APIError.httpError(statusCode: httpResponse.statusCode, message: nil)
        }
    }
}
```

### Endpoint Definitions

```swift
enum UserEndpoint: Endpoint {
    case list
    case get(id: String)
    case create(CreateUserRequest)
    case update(id: String, UpdateUserRequest)
    case delete(id: String)

    var path: String {
        switch self {
        case .list:
            return "/users"
        case .get(let id), .update(let id, _), .delete(let id):
            return "/users/\(id)"
        case .create:
            return "/users"
        }
    }

    var method: HTTPMethod {
        switch self {
        case .list, .get:
            return .get
        case .create:
            return .post
        case .update:
            return .patch
        case .delete:
            return .delete
        }
    }

    var body: Encodable? {
        switch self {
        case .create(let request):
            return request
        case .update(_, let request):
            return request
        default:
            return nil
        }
    }
}
```

### Usage in ViewModel

```swift
@Observable
class UserListViewModel {
    var users: [User] = []
    var isLoading = false
    var error: Error?

    private let apiClient: APIClient

    init(apiClient: APIClient = .shared) {
        self.apiClient = apiClient
    }

    func loadUsers() async {
        isLoading = true
        defer { isLoading = false }

        do {
            users = try await apiClient.request(UserEndpoint.list)
        } catch {
            self.error = error
        }
    }

    func deleteUser(_ user: User) async {
        do {
            try await apiClient.requestVoid(UserEndpoint.delete(id: user.id.uuidString))
            users.removeAll { $0.id == user.id }
        } catch {
            self.error = error
        }
    }
}
```

## Token Refresh

```swift
actor APIClient {
    private var refreshTask: Task<String, Error>?

    private func refreshTokenIfNeeded() async throws -> String {
        if let refreshTask {
            return try await refreshTask.value
        }

        guard let refreshToken = tokenManager.refreshToken else {
            throw APIError.unauthorized
        }

        let task = Task {
            defer { refreshTask = nil }

            let request = RefreshTokenRequest(refreshToken: refreshToken)
            let response: AuthResponse = try await performUnauthenticatedRequest(
                AuthEndpoint.refresh(request)
            )

            tokenManager.accessToken = response.accessToken
            tokenManager.refreshToken = response.refreshToken

            return response.accessToken
        }

        refreshTask = task
        return try await task.value
    }
}
```

## Download & Upload

### Download with Progress

```swift
func downloadFile(from url: URL, to destination: URL) async throws {
    let (tempURL, response) = try await URLSession.shared.download(from: url)

    guard let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 200 else {
        throw APIError.invalidResponse
    }

    try FileManager.default.moveItem(at: tempURL, to: destination)
}

// With progress (use URLSessionDownloadDelegate)
class DownloadManager: NSObject, URLSessionDownloadDelegate {
    var progressHandler: ((Double) -> Void)?

    func urlSession(
        _ session: URLSession,
        downloadTask: URLSessionDownloadTask,
        didWriteData bytesWritten: Int64,
        totalBytesWritten: Int64,
        totalBytesExpectedToWrite: Int64
    ) {
        let progress = Double(totalBytesWritten) / Double(totalBytesExpectedToWrite)
        progressHandler?(progress)
    }

    func urlSession(
        _ session: URLSession,
        downloadTask: URLSessionDownloadTask,
        didFinishDownloadingTo location: URL
    ) {
        // Handle completed download
    }
}
```

### Upload

```swift
func uploadImage(_ imageData: Data) async throws -> UploadResponse {
    var request = URLRequest(url: URL(string: "https://api.example.com/upload")!)
    request.httpMethod = "POST"

    let boundary = UUID().uuidString
    request.setValue("multipart/form-data; boundary=\(boundary)", forHTTPHeaderField: "Content-Type")

    var body = Data()
    body.append("--\(boundary)\r\n".data(using: .utf8)!)
    body.append("Content-Disposition: form-data; name=\"file\"; filename=\"image.jpg\"\r\n".data(using: .utf8)!)
    body.append("Content-Type: image/jpeg\r\n\r\n".data(using: .utf8)!)
    body.append(imageData)
    body.append("\r\n--\(boundary)--\r\n".data(using: .utf8)!)

    request.httpBody = body

    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(UploadResponse.self, from: data)
}
```

## Request/Response DTOs

```swift
// Requests
struct CreateUserRequest: Encodable {
    let name: String
    let email: String
    let password: String
}

struct UpdateUserRequest: Encodable {
    let name: String?
    let email: String?
}

// Responses
struct User: Codable, Identifiable {
    let id: UUID
    let name: String
    let email: String
    let createdAt: Date
}

struct PaginatedResponse<T: Codable>: Codable {
    let data: [T]
    let page: Int
    let totalPages: Int
    let totalItems: Int
}

struct ErrorResponse: Codable {
    let code: String
    let message: String
}
```

## Best Practices

1. **Use async/await** - Not completion handlers
2. **Actor for API client** - Thread-safe token management
3. **Type-safe endpoints** - Enums over string URLs
4. **Centralized error handling** - Consistent error types
5. **Token refresh** - Handle 401 gracefully
6. **Request timeout** - Set reasonable timeouts
7. **Cancel tasks** - Cancel on view disappear
