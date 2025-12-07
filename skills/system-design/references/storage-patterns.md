# Storage Patterns

## Quick Decision Matrix

| If you need... | Use | Avoid |
|----------------|-----|-------|
| User uploads (images, documents) | Cloud storage (S3) | Local file system |
| Public static assets | CDN | Serving from app server |
| Large media files (video) | Cloud storage + CDN | Database |
| Temporary files | Signed URLs + cleanup | Permanent storage |
| iOS image caching | URLCache + disk cache | Re-downloading |

---

## Cloud Object Storage

| Service | Best For | Pros | Cons |
|---------|----------|------|------|
| AWS S3 | General purpose, AWS ecosystem | Mature, scalable, many features | AWS lock-in |
| Google Cloud Storage | GCP ecosystem, ML workflows | Good performance, GCP integration | Smaller ecosystem |
| Cloudflare R2 | Cost optimization, no egress fees | S3-compatible, cheap | Newer service |
| DigitalOcean Spaces | Simple needs, budget | S3-compatible, simple pricing | Fewer features |

### S3 Best Practices
- Use appropriate storage class (Standard, Infrequent Access, Glacier)
- Enable versioning for important buckets
- Set lifecycle rules for cleanup
- Use server-side encryption
- Implement proper IAM policies

---

## Upload Patterns

### Direct Upload (Recommended)

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ iOS App │────▶│ Backend │────▶│   S3    │
└─────────┘     └─────────┘     └─────────┘
     │               │               ▲
     │  1. Request   │               │
     │  presigned    │               │
     │  URL          │  2. Generate  │
     │◀──────────────│  presigned    │
     │               │  URL          │
     │                               │
     │  3. Upload directly ──────────┘
     │
     │  4. Confirm upload
     │──────────────▶│
```

**Benefits**:
- Reduces backend bandwidth
- Faster uploads (direct to storage)
- Backend not bottleneck

### Presigned URL Generation (Backend)
```python
import boto3
from datetime import timedelta

s3 = boto3.client('s3')

def generate_upload_url(filename: str, content_type: str) -> dict:
    key = f"uploads/{uuid4()}/{filename}"

    url = s3.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': 'my-bucket',
            'Key': key,
            'ContentType': content_type
        },
        ExpiresIn=3600  # 1 hour
    )

    return {
        'upload_url': url,
        'key': key,
        'expires_in': 3600
    }
```

### iOS Upload Implementation
```swift
func uploadFile(data: Data, to presignedURL: URL, contentType: String) async throws {
    var request = URLRequest(url: presignedURL)
    request.httpMethod = "PUT"
    request.setValue(contentType, forHTTPHeaderField: "Content-Type")

    let (_, response) = try await URLSession.shared.upload(for: request, from: data)

    guard let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 200 else {
        throw UploadError.failed
    }
}
```

---

## CDN (Content Delivery Network)

| CDN | Best For | Features |
|-----|----------|----------|
| CloudFront | AWS ecosystem | S3 integration, Lambda@Edge |
| Cloudflare | Performance, DDoS protection | Edge workers, generous free tier |
| Fastly | Real-time purging | Instant cache invalidation |
| Bunny CDN | Cost-effective | Simple, global |

### CDN Architecture
```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ iOS App │────▶│   CDN   │────▶│ Origin  │
└─────────┘     │ (Edge)  │     │  (S3)   │
                └─────────┘     └─────────┘
                     │
              Cache if present
              Fetch & cache if not
```

### When to Use CDN
- Static assets (images, videos, CSS, JS)
- User-uploaded content accessed globally
- API responses that can be cached
- High-traffic media files

### CDN Configuration Tips
- Set appropriate cache headers
- Use cache invalidation for updates
- Configure custom domain with SSL
- Set up origin failover

---

## Image Handling

### Image Processing Pipeline
```
Upload → Process → Store → Serve via CDN

Processing:
- Resize to multiple sizes (thumbnail, medium, full)
- Convert format (WebP for web, HEIC preserved for iOS)
- Strip metadata (privacy)
- Optimize quality
```

### Image Variants Strategy
| Variant | Size | Use Case |
|---------|------|----------|
| thumbnail | 150x150 | Lists, grids |
| medium | 600x600 | Detail views |
| large | 1200x1200 | Full screen |
| original | As uploaded | Download |

### On-Demand Resizing
Use services like:
- Cloudflare Images
- Imgix
- Cloudinary
- AWS Lambda + Sharp

```
https://cdn.example.com/image.jpg?width=300&height=300&fit=cover
```

---

## iOS Image Caching

### URLCache (Built-in)
```swift
// Configure cache
let cache = URLCache(
    memoryCapacity: 50_000_000,  // 50MB
    diskCapacity: 100_000_000    // 100MB
)
URLCache.shared = cache
```

### Kingfisher/SDWebImage Pattern
```swift
// Using Kingfisher
imageView.kf.setImage(
    with: url,
    placeholder: UIImage(named: "placeholder"),
    options: [
        .transition(.fade(0.2)),
        .cacheOriginalImage
    ]
)
```

### Cache Strategy

| Image Type | Cache Duration | Storage |
|------------|----------------|---------|
| Profile photos | 1 day | Memory + Disk |
| Product images | 1 week | Memory + Disk |
| Static assets | 1 month | Disk only |
| Thumbnails | 1 day | Memory |

---

## Video Storage

### Video Handling Considerations
- Store original + transcoded versions
- Use adaptive bitrate streaming (HLS)
- Serve via CDN
- Consider dedicated video platforms (Mux, Cloudflare Stream)

### HLS Streaming
```
video/
├── original.mp4
├── hls/
│   ├── master.m3u8
│   ├── 1080p/
│   ├── 720p/
│   └── 480p/
```

### iOS Video Playback
```swift
import AVKit

let player = AVPlayer(url: hlsURL)
let playerViewController = AVPlayerViewController()
playerViewController.player = player
present(playerViewController, animated: true) {
    player.play()
}
```

---

## File Organization

### S3 Bucket Structure
```
bucket/
├── uploads/           # User uploads (private)
│   └── {user_id}/
│       └── {uuid}/
│           └── filename.jpg
├── public/            # Public assets
│   ├── images/
│   └── documents/
├── processed/         # Processed files
│   └── images/
│       └── {uuid}/
│           ├── thumb.jpg
│           ├── medium.jpg
│           └── large.jpg
└── temp/              # Temporary (auto-cleanup)
```

### File Naming
```
# Good: Unique, no conflicts
uploads/123/550e8400-e29b-41d4-a716-446655440000/profile.jpg

# Bad: Conflicts, special characters
uploads/john's photo (1).jpg
```

---

## Security

### Access Control
| Content Type | Access | Implementation |
|--------------|--------|----------------|
| User private files | Signed URLs | Time-limited, per-user |
| Public assets | Public bucket/CDN | Cache-friendly |
| Sensitive documents | Backend proxy | Never direct access |

### Signed URL Pattern
```python
def get_download_url(key: str, user_id: str) -> str:
    # Verify user has access
    if not user_can_access(user_id, key):
        raise PermissionDenied()

    return s3.generate_presigned_url(
        'get_object',
        Params={'Bucket': 'my-bucket', 'Key': key},
        ExpiresIn=3600
    )
```

---

## Storage Checklist

- [ ] Cloud storage provider selected (S3, R2, etc.)
- [ ] CDN configured for public assets
- [ ] Direct upload with presigned URLs implemented
- [ ] Image processing pipeline (resize, optimize)
- [ ] iOS image caching strategy (Kingfisher/native)
- [ ] File organization structure defined
- [ ] Access control (signed URLs for private content)
- [ ] Lifecycle rules for cleanup
- [ ] Backup strategy for critical data
- [ ] Cost monitoring and optimization
