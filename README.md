# Forum Uygulaması - Backend Projesi

Merhaba! Bu proje seni Java Spring Boot, MongoDB ve Docker dünyasına sokacak. Gerçek hayatta karşına çıkacak senaryolara benzer bir uygulama geliştireceğiz: basit bir forum sistemi.

---

## Ne Yapacaksın?

Bir forum uygulaması yazacaksın. Kullanıcılar sisteme kayıt olacak, konu açabilecek ve bu konulara yorum yapabilecek. Klasik bir forum düşün: Technopat, DonanımHaber gibi sitelerin çok basitleştirilmiş hali.

Backend tarafını Spring Boot ile yazacaksın. Veritabanı olarak MongoDB kullanacaksın (SQL değil, NoSQL). Frontend tarafında karışık bir şey beklemiyorum, basit HTML/CSS/JavaScript yeterli. Son olarak tüm bu yapıyı Docker container'larına koyup docker-compose ile ayağa kaldıracaksın.

---

## Neden Bu Teknolojiler?

**Spring Boot** şu an enterprise Java dünyasının standartı. Şirketlerin büyük çoğunluğu Spring ekosistemini kullanıyor. Dependency Injection, REST API geliştirme, veritabanı entegrasyonu gibi konuları öğreneceksin.

**MongoDB** NoSQL dünyasını görmeni istiyorum. Her yerde SQL kullanılmıyor, document-based veritabanlarının nasıl çalıştığını anlamak önemli. Ayrıca Spring Data MongoDB ile çalışmak klasik JPA'dan biraz farklı, bu deneyimi kazanman iyi olur.

**Docker** artık olmazsa olmaz. Bugün bir yazılımcının "benim makinemde çalışıyordu" demesi kabul edilmiyor. Uygulamanı container'a koyup her yerde aynı şekilde çalışmasını sağlayacaksın.

---

## Proje Yapısı

Projeyi şu şekilde organize etmeni istiyorum:

```
forum-app/
│
├── backend/
│   ├── src/
│   │   └── main/
│   │       ├── java/
│   │       │   └── com/
│   │       │       └── forum/
│   │       │           ├── model/
│   │       │           │   ├── User.java
│   │       │           │   ├── Topic.java
│   │       │           │   └── Comment.java
│   │       │           │
│   │       │           ├── repository/
│   │       │           │   ├── UserRepository.java
│   │       │           │   ├── TopicRepository.java
│   │       │           │   └── CommentRepository.java
│   │       │           │
│   │       │           ├── service/
│   │       │           │   ├── UserService.java
│   │       │           │   ├── TopicService.java
│   │       │           │   └── CommentService.java
│   │       │           │
│   │       │           ├── controller/
│   │       │           │   ├── UserController.java
│   │       │           │   ├── TopicController.java
│   │       │           │   └── CommentController.java
│   │       │           │
│   │       │           └── ForumApplication.java
│   │       │
│   │       └── resources/
│   │           └── application.properties
│   │
│   ├── pom.xml
│   └── Dockerfile
│
├── frontend/
│   ├── index.html
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   └── app.js
│   └── Dockerfile
│
├── docker-compose.yml
└── README.md
```

Bu yapıya **layered architecture** deniyor. Her katmanın bir görevi var:

- **Model**: Veritabanındaki tablolarını temsil eden Java sınıfları
- **Repository**: Veritabanı işlemlerini yapan katman (CRUD operasyonları)
- **Service**: İş mantığının yazıldığı katman
- **Controller**: HTTP isteklerini karşılayan ve response dönen katman

Controller doğrudan Repository'yi çağırmasın, arada Service katmanı olsun. Bu şekilde kod daha düzenli ve test edilebilir oluyor.

---

## Veri Modelleri

Üç tane model sınıfın olacak. Bunları detaylı anlatayım:

### User (Kullanıcı)

```java
@Document(collection = "users")
public class User {
    
    @Id
    private String id;
    
    private String username;
    
    private String email;
    
    private LocalDateTime createdAt;
    
    // getter, setter, constructor
}
```

Dikkat etmen gerekenler:
- `@Document` annotation'ı bu sınıfın MongoDB'de bir collection'a karşılık geldiğini söylüyor
- `@Id` annotation'ı primary key'i belirtiyor. MongoDB'de id'ler String tipinde ve otomatik oluşturuluyor (ObjectId)
- `username` ve `email` unique olmalı, aynı kullanıcı adı veya email ile iki kayıt olmamalı
- `createdAt` alanını kayıt oluşturulurken otomatik set et

### Topic (Konu)

```java
@Document(collection = "topics")
public class Topic {
    
    @Id
    private String id;
    
    private String title;
    
    private String content;
    
    private String authorId;
    
    private String authorUsername;
    
    private LocalDateTime createdAt;
    
    private LocalDateTime updatedAt;
    
    // getter, setter, constructor
}
```

Burada bir tasarım kararı var: `authorId` ve `authorUsername` alanlarını ayrı tuttum. Neden?

MongoDB'de join işlemi yoktur (aslında $lookup var ama performanslı değil). Bu yüzden sık kullanılan verileri denormalize ederiz. Konu listelerken her seferinde User collection'ına gidip kullanıcı adını çekmek yerine, konuyu kaydederken kullanıcı adını da yanına yazıyoruz. Bu NoSQL dünyasında yaygın bir pattern.

Validation kuralları:
- `title` minimum 5, maksimum 200 karakter olsun
- `content` minimum 20 karakter olsun (çok kısa konu açılmasın)
- `authorId` zorunlu, boş bırakılamaz

### Comment (Yorum)

```java
@Document(collection = "comments")
public class Comment {
    
    @Id
    private String id;
    
    private String content;
    
    private String topicId;
    
    private String authorId;
    
    private String authorUsername;
    
    private LocalDateTime createdAt;
    
    // getter, setter, constructor
}
```

Yorum modeli Topic'e benziyor. `topicId` alanı yorumun hangi konuya ait olduğunu gösteriyor.

Validation:
- `content` minimum 2 karakter (çok kısa yorum olmasın ama emoji atabilsin)
- `topicId` ve `authorId` zorunlu

---

## API Tasarımı

REST API tasarlarken bazı kurallara uymanı istiyorum. Bunlar endüstri standardı:

- URL'lerde fiil kullanma, isim kullan: `/getUsers` değil `/users`
- HTTP metodları işlemi belirler: GET okuma, POST oluşturma, PUT güncelleme, DELETE silme
- Çoğul isim kullan: `/user` değil `/users`
- İlişkili kaynaklar için nested URL kullan: `/topics/123/comments`

### Kullanıcı Endpoint'leri

**POST /api/users** - Yeni kullanıcı oluşturma

Request body:
```json
{
    "username": "ahmet_dev",
    "email": "ahmet@example.com"
}
```

Response (201 Created):
```json
{
    "id": "6579a8b2c4f5e8d3b2a1c0f9",
    "username": "ahmet_dev",
    "email": "ahmet@example.com",
    "createdAt": "2024-01-15T10:30:00"
}
```

Hata durumları:
- Username zaten varsa: 409 Conflict
- Email zaten varsa: 409 Conflict
- Geçersiz email formatı: 400 Bad Request

**GET /api/users** - Tüm kullanıcıları listeleme

Response (200 OK):
```json
[
    {
        "id": "6579a8b2c4f5e8d3b2a1c0f9",
        "username": "ahmet_dev",
        "email": "ahmet@example.com",
        "createdAt": "2024-01-15T10:30:00"
    },
    {
        "id": "6579a8b2c4f5e8d3b2a1c0fa",
        "username": "mehmet_42",
        "email": "mehmet@example.com",
        "createdAt": "2024-01-15T11:45:00"
    }
]
```

**GET /api/users/{id}** - Tek kullanıcı detayı

Response (200 OK): Yukarıdaki gibi tek bir user objesi

Hata durumu:
- Kullanıcı bulunamazsa: 404 Not Found

### Konu Endpoint'leri

**POST /api/topics** - Yeni konu oluşturma

Request body:
```json
{
    "title": "Spring Boot ile REST API Geliştirme",
    "content": "Merhaba arkadaşlar, Spring Boot öğrenmeye başladım ve REST API konusunda kafama takılan birkaç soru var. Öncelikle Controller ile RestController arasındaki fark nedir? İkisi de aynı işi yapıyor gibi görünüyor...",
    "authorId": "6579a8b2c4f5e8d3b2a1c0f9"
}
```

Response (201 Created):
```json
{
    "id": "6579b1c3d5e6f7a8b9c0d1e2",
    "title": "Spring Boot ile REST API Geliştirme",
    "content": "Merhaba arkadaşlar, Spring Boot öğrenmeye başladım...",
    "authorId": "6579a8b2c4f5e8d3b2a1c0f9",
    "authorUsername": "ahmet_dev",
    "createdAt": "2024-01-15T14:20:00",
    "updatedAt": "2024-01-15T14:20:00"
}
```

Dikkat: Request'te `authorUsername` yok, sadece `authorId` var. Service katmanında User'ı bulup username'i oradan alacaksın.

Hata durumları:
- authorId geçersizse (kullanıcı yoksa): 400 Bad Request
- title çok kısaysa: 400 Bad Request

**GET /api/topics** - Tüm konuları listeleme

Bu endpoint'i biraz akıllı yap. Varsayılan olarak en son açılan konular üstte olsun (createdAt'e göre descending sırala).

Response (200 OK):
```json
[
    {
        "id": "6579b1c3d5e6f7a8b9c0d1e2",
        "title": "Spring Boot ile REST API Geliştirme",
        "content": "Merhaba arkadaşlar, Spring Boot öğrenmeye başladım...",
        "authorId": "6579a8b2c4f5e8d3b2a1c0f9",
        "authorUsername": "ahmet_dev",
        "createdAt": "2024-01-15T14:20:00",
        "updatedAt": "2024-01-15T14:20:00"
    }
]
```

**GET /api/topics/{id}** - Konu detayı

Tek bir topic döner. 404 Not Found durumunu handle et.

**PUT /api/topics/{id}** - Konu güncelleme

Request body:
```json
{
    "title": "Spring Boot ile REST API Geliştirme (Güncellendi)",
    "content": "Merhaba arkadaşlar, Spring Boot öğrenmeye başladım... EDIT: Sorunum çözüldü, teşekkürler!"
}
```

Güncelleme yapıldığında `updatedAt` alanını güncellemeyi unutma.

Hata durumları:
- Konu bulunamazsa: 404 Not Found
- Geçersiz veri: 400 Bad Request

**DELETE /api/topics/{id}** - Konu silme

Response: 204 No Content (body boş döner)

Önemli: Bir konu silindiğinde o konuya ait tüm yorumlar da silinmeli. Bunu Service katmanında handle et.

### Yorum Endpoint'leri

**POST /api/topics/{topicId}/comments** - Konuya yorum ekleme

Request body:
```json
{
    "content": "Çok güzel anlatmışsın, teşekkürler!",
    "authorId": "6579a8b2c4f5e8d3b2a1c0fa"
}
```

Response (201 Created):
```json
{
    "id": "6579c2d4e5f6a7b8c9d0e1f2",
    "content": "Çok güzel anlatmışsın, teşekkürler!",
    "topicId": "6579b1c3d5e6f7a8b9c0d1e2",
    "authorId": "6579a8b2c4f5e8d3b2a1c0fa",
    "authorUsername": "mehmet_42",
    "createdAt": "2024-01-15T15:30:00"
}
```

Hata durumları:
- Topic bulunamazsa: 404 Not Found
- authorId geçersizse: 400 Bad Request

**GET /api/topics/{topicId}/comments** - Konunun yorumlarını listeleme

Yorumları createdAt'e göre ascending sırala (eski yorumlar üstte, yeni yorumlar altta - forum mantığı).

**DELETE /api/comments/{id}** - Yorum silme

Response: 204 No Content

---

## Service Katmanı Detayları

Service katmanında iş mantığını yazacaksın. Birkaç örnek vereyim:

### TopicService

```java
@Service
public class TopicService {
    
    private final TopicRepository topicRepository;
    private final UserRepository userRepository;
    private final CommentRepository commentRepository;
    
    // Constructor injection kullan, @Autowired field injection yerine
    public TopicService(TopicRepository topicRepository, 
                        UserRepository userRepository,
                        CommentRepository commentRepository) {
        this.topicRepository = topicRepository;
        this.userRepository = userRepository;
        this.commentRepository = commentRepository;
    }
    
    public Topic createTopic(String title, String content, String authorId) {
        // 1. Önce user'ı bul
        User author = userRepository.findById(authorId)
            .orElseThrow(() -> new RuntimeException("Kullanıcı bulunamadı"));
        
        // 2. Topic nesnesini oluştur
        Topic topic = new Topic();
        topic.setTitle(title);
        topic.setContent(content);
        topic.setAuthorId(authorId);
        topic.setAuthorUsername(author.getUsername());
        topic.setCreatedAt(LocalDateTime.now());
        topic.setUpdatedAt(LocalDateTime.now());
        
        // 3. Kaydet ve döndür
        return topicRepository.save(topic);
    }
    
    public void deleteTopic(String topicId) {
        // 1. Topic var mı kontrol et
        if (!topicRepository.existsById(topicId)) {
            throw new RuntimeException("Konu bulunamadı");
        }
        
        // 2. Önce bu konuya ait tüm yorumları sil
        commentRepository.deleteByTopicId(topicId);
        
        // 3. Sonra konuyu sil
        topicRepository.deleteById(topicId);
    }
    
    // Diğer metodlar...
}
```

Dikkat etmen gerekenler:

1. **Constructor Injection**: `@Autowired` ile field injection yerine constructor injection kullan. Bu test yazarken mock'lamayı kolaylaştırır ve dependency'leri explicit yapar.

2. **Exception Handling**: Şimdilik basit RuntimeException fırlatabilirsin ama ideal olanda custom exception sınıfları oluşturursun (örn: `UserNotFoundException`, `TopicNotFoundException`).

3. **Transaction**: MongoDB single document işlemlerinde transaction'a gerek yok ama deleteTopic gibi birden fazla collection'ı etkileyen işlemlerde dikkatli ol.

---

## Exception Handling

Controller'larda exception'ları düzgün handle etmen lazım. Bunun için `@ControllerAdvice` kullanabilirsin:

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<Map<String, String>> handleRuntimeException(RuntimeException ex) {
        Map<String, String> error = new HashMap<>();
        error.put("error", ex.getMessage());
        return ResponseEntity.badRequest().body(error);
    }
}
```

Bu sayede her controller'da ayrı ayrı try-catch yazmana gerek kalmaz. Fırlattığın exception'lar burada yakalanır ve uygun HTTP response'a dönüştürülür.

---

## MongoDB Bağlantısı

`application.properties` dosyası şöyle olacak:

```properties
spring.application.name=forum-backend

# MongoDB bağlantısı
spring.data.mongodb.uri=mongodb://localhost:27017/forumdb

# Server port
server.port=8080
```

Docker kullanırken `localhost` yerine container adını yazacaksın. Bunu environment variable ile çözeceğiz:

```properties
spring.data.mongodb.uri=${MONGODB_URI:mongodb://localhost:27017/forumdb}
```

Bu syntax şunu söylüyor: "MONGODB_URI environment variable'ı varsa onu kullan, yoksa default olarak localhost'u kullan." Böylece local'de çalışırken localhost'a, Docker'da çalışırken container'a bağlanır.

---

## Frontend

Frontend tarafını çok karmaşık yapmanı beklemiyorum. Amacımız backend ile iletişimi öğrenmek. Basit HTML/CSS/JavaScript yeterli.

### Sayfalar

**Ana Sayfa (index.html)**
- Konu listesi görüntülenir
- Her konunun başlığı, yazarı ve tarihi gözükür
- "Yeni Konu Aç" butonu olur
- Konuya tıklayınca detay sayfasına gider

**Konu Detay Sayfası (topic.html)**
- Konunun tam içeriği görüntülenir
- Altında yorumlar listelenir
- Yorum yazma formu olur

**Kullanıcı Seçimi**
- Basit bir dropdown ile kullanıcı seçimi yapılabilir
- Ya da sayfanın bir köşesinde "Kullanıcı adı gir" input'u olur
- Authentication yapmayacağız, sadece kullanıcı adı ile işlem yapacağız

### API İletişimi

Fetch API kullan:

```javascript
// Konuları çekme
async function loadTopics() {
    try {
        const response = await fetch('http://localhost:8080/api/topics');
        const topics = await response.json();
        displayTopics(topics);
    } catch (error) {
        console.error('Konular yüklenirken hata:', error);
    }
}

// Yeni konu oluşturma
async function createTopic(title, content, authorId) {
    try {
        const response = await fetch('http://localhost:8080/api/topics', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ title, content, authorId })
        });
        
        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.message);
        }
        
        const newTopic = await response.json();
        return newTopic;
    } catch (error) {
        console.error('Konu oluşturulurken hata:', error);
        throw error;
    }
}
```

### CORS Ayarı

Frontend ayrı port'ta çalışacağı için CORS hatası alacaksın. Backend'de bunu çözmen lazım:

```java
@Configuration
public class CorsConfig {
    
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("http://localhost:3000")
                    .allowedMethods("GET", "POST", "PUT", "DELETE")
                    .allowedHeaders("*");
            }
        };
    }
}
```

---

## Docker Kurulumu

### Backend Dockerfile

```dockerfile
# Build aşaması
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Run aşaması
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Bu multi-stage build. İlk stage'de Maven ile projeyi build ediyoruz, ikinci stage'de sadece JAR dosyasını alıp çalıştırıyoruz. Bu sayede final image çok daha küçük oluyor.

### Frontend Dockerfile

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
```

Frontend için basit bir nginx container'ı yeterli. Static dosyaları nginx serve edecek.

### docker-compose.yml

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7
    container_name: forum-mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    networks:
      - forum-network

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: forum-backend
    ports:
      - "8080:8080"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/forumdb
    depends_on:
      - mongodb
    networks:
      - forum-network

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: forum-frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - forum-network

networks:
  forum-network:
    driver: bridge

volumes:
  mongodb_data:
```

Burada birkaç önemli nokta var:

1. **networks**: Tüm container'lar aynı network'te. Bu sayede birbirlerini isimle bulabiliyorlar (mongodb, backend, frontend).

2. **depends_on**: Backend, MongoDB'ye bağımlı. Frontend, Backend'e bağımlı. Docker Compose bu sırayla başlatır.

3. **volumes**: MongoDB verisi volume'da saklanıyor. Container silinse bile veri kaybolmuyor.

4. **environment**: Backend'e MONGODB_URI'yi environment variable olarak geçiyoruz.

### Çalıştırma

```bash
# Tüm servisleri başlat
docker-compose up -d

# Logları izle
docker-compose logs -f

# Servisleri durdur
docker-compose down

# Servisleri durdur ve volume'ları da sil
docker-compose down -v
```

---

## Test Etme

### Postman ile API Testi

Postman kullanarak API'ni test et. Şu sırayla dene:

1. Birkaç kullanıcı oluştur (POST /api/users)
2. Kullanıcıları listele (GET /api/users)
3. Bir konu oluştur (POST /api/topics)
4. Konuları listele (GET /api/topics)
5. Konuya yorum yap (POST /api/topics/{id}/comments)
6. Yorumları listele (GET /api/topics/{id}/comments)
7. Konuyu güncelle (PUT /api/topics/{id})
8. Yorumu sil (DELETE /api/comments/{id})
9. Konuyu sil (DELETE /api/topics/{id}) - yorumların da silindiğini kontrol et

Her endpoint için hem başarılı hem hatalı senaryoları test et. Örneğin:
- Var olmayan bir kullanıcı ile konu açmayı dene
- Aynı username ile iki kullanıcı oluşturmayı dene
- Çok kısa başlık ile konu açmayı dene

### Postman Collection

Test ettiğin tüm request'leri Postman Collection olarak kaydet ve projeyle birlikte paylaş. Bu hem senin için dokümantasyon olur hem de başkalarının projeyi test etmesini kolaylaştırır.

---

## Teslim

Projeyi teslim ederken şunlar olmalı:

1. **GitHub Repository**: Kodun tamamı GitHub'da public repo olarak

2. **README.md**: Şunları içermeli:
   - Projenin kısa açıklaması
   - Kullanılan teknolojiler
   - Kurulum adımları (local ve Docker)
   - API endpoint'lerinin listesi
   - Screenshot'lar (opsiyonel ama güzel olur)

3. **docker-compose up çalışmalı**: Repo'yu clone'layıp docker-compose up dedikten sonra her şey çalışmalı

4. **Postman Collection**: API testleri için export edilmiş JSON dosyası

---

## Bonus Özellikler (Opsiyonel)

Eğer erken bitirirsen veya kendini zorlamak istersen şunları ekleyebilirsin:

1. **Pagination**: Konu ve yorum listelerinde sayfalama. Query parameter olarak `page` ve `size` al.

2. **Arama**: Konu başlığında arama yapabilme. GET /api/topics?search=spring

3. **Sıralama**: Konuları farklı kriterlere göre sıralama. GET /api/topics?sort=createdAt,desc

4. **Validation**: Spring Validation kullanarak input validasyonu. @NotBlank, @Size, @Email gibi annotation'lar.

5. **DTO Pattern**: Entity'leri doğrudan dönmek yerine DTO (Data Transfer Object) kullanmak.

6. **Swagger/OpenAPI**: API dokümantasyonu için Swagger entegrasyonu.

Bunlar zorunlu değil ama yaparsan artı puan.

---

## Son Notlar

- Takıldığın yerde önce Google'la, Stack Overflow'a bak
- Resmi dokümantasyonları oku (Spring, MongoDB, Docker)
- Hata mesajlarını dikkatlice oku, genelde çözümü söylüyorlar
- Commit'lerini düzenli at, her özellik için ayrı commit
- Kod yazarken yorum ekle, ne yaptığını açıkla

