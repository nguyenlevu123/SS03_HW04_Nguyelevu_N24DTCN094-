# BÁO CÁO BÀI TẬP: SÁNG TẠO — MODULE ETL RESUME PARSER (RIKKEI ACADEMY HR)

* **Học viên:** Nguyễn lê Vũ
* **Mã số sinh viên:** N24DTCN094
* **Môn học:** Kỹ năng ứng dụng AI

---

## PHẦN 1: TIÊU ĐỀ BÀI TẬP VÀ YÊU CẦU ĐỀ BÀI

### Tiêu đề
**Bài 4: Sáng tạo — Module ETL Resume Parser (Rikkei Academy HR)**

### Bối cảnh & Yêu cầu đề bài
Bộ phận nhân sự của Rikkei Academy hàng ngày phải tiếp nhận hàng trăm CV ứng viên dưới dạng văn bản thô phi cấu trúc từ nhiều nguồn khác nhau. Để giảm tải công việc nhập liệu thủ công, hệ thống quản lý HR cần phát triển một module ETL (Extract - Transform - Load) sử dụng AI:
- **Extract**: Nhận văn bản thô từ CV ứng viên.
- **Transform**: Gọi LLM thông qua ChatModel kết hợp BeanOutputConverter để trích xuất dữ liệu thô thành một Java Record có cấu trúc.
- **Load**: Kiểm tra tính hợp lệ của dữ liệu (validation) và lưu trữ vào cơ sở dữ liệu SQL thông qua Spring Data JPA.

### Yêu cầu cụ thể
1. Định nghĩa Java Record `CandidateExtraction` để bóc tách thông tin ứng viên bao gồm: `fullName`, `phone`, `email`, `skills` (List<String>), `yearsExperience`.
2. Thiết kế thực thể JPA `@Entity Candidate` tương ứng và interface `CandidateRepository` (kế thừa từ `JpaRepository`).
3. Viết mã nguồn lớp `@Service CandidateETLService` triển khai phương thức `processResume(String resumeText)`. Trong Service, bắt buộc thực hiện tối thiểu 02 bước kiểm tra dữ liệu hợp lệ trước khi lưu xuống database.
4. Vẽ sơ đồ ASCII mô tả trực quan luồng đi của dữ liệu từ khi nhận CV thô cho đến khi lưu trữ thành công vào Database.
5. Phân tích chi tiết trade-off (ưu và nhược điểm) của việc đặt lệnh gọi API LLM (qua ChatModel) bên trong so với bên ngoài phạm vi của một phương thức có đánh dấu `@Transactional` liên quan đến việc chiếm dụng kết nối cơ sở dữ liệu (connection pooling) và xử lý rollback khi có lỗi xảy ra.

---

## PHẦN 2: BÁO CÁO CUỘC TRÒ CHUYỆN THỰC TẾ VỚI AI VÀ GIẢI PHÁP CHI TIẾT

### 1. SƠ ĐỒ LUỒNG DỮ LIỆU ETL (ASCII DIAGRAM)

```text
+---------------------------------------------------------------------------------------------------------+
|                                    LUỒNG ĐI CỦA DỮ LIỆU ETL RESUME PARSER                               |
+---------------------------------------------------------------------------------------------------------+
|                                                                                                         |
|  [ CV Văn bản thô ] (Resume Text Input)                                                                 |
|         ||                                                                                              |
|         \/                                                                                              |
|  +----------------------------------+                                                                   |
|  |      CandidateETLService         | (Phạm vi ngoài TRANSACTION - Không chiếm dụng Connection DB)       |
|  |  (Extract & Transform Phase)     |                                                                   |
|  |                                  |                                                                   |
|  |  1. Nhận resumeText và cấu hình  |                                                                   |
|  |     BeanOutputConverter          |                                                                   |
|  |  2. Tạo Prompt chứa format       |                                                                   |
|  |  3. Gọi LLM API (Mạng ngoài)     | =====[ Network Call ]=====> [ LLM / GPT Engine ]                 |
|  |                                  | <====[ JSON Response ]===== [ Trả về dữ liệu thô ]               |
|  |  4. Chuyển đổi JSON thu được     |                                                                   |
|  |     thành Java Record DTO        |                                                                   |
|  +----------------------------------+                                                                   |
|         ||                                                                                              |
|         \/ (Dữ liệu dạng: CandidateExtraction Record)                                                   |
|  +----------------------------------+                                                                   |
|  |   BỘ LỌC KIỂM TRA HỢP LỆ (VAL)   |                                                                   |
|  |  - Kiểm tra họ tên trống?        | (Nghiệp vụ ứng dụng - Ném Exception ngay nếu dữ liệu rác)         |
|  |  - Kiểm tra định dạng Email?     |                                                                   |
|  |  - Kiểm tra năm kinh nghiệm >= 0?|                                                                   |
|  +----------------------------------+                                                                   |
|         ||                                                                                              |
|         \/ (Dữ liệu hợp lệ hợp pháp)                                                                    |
|  +----------------------------------+                                                                   |
|  |   @Transactional Save Method     | <==== Bắt đầu chiếm dụng 1 Connection từ HikariCP Pool            |
|  |  (Load Phase - DB Write)         |                                                                   |
|  |                                  |                                                                   |
|  |  1. Kiểm tra trùng lặp Email     | =====[ Query SELECT ]=====> [ Candidate Table (DB) ]             |
|  |  2. Map Record -> JPA Entity     |                                                                   |
|  |  3. Lưu xuống Database           | =====[ Query INSERT ]=====> [ PostgreSQL / MySQL ]               |
|  +----------------------------------+                                                                   |
|         ||                                                                                              |
|         \/ (Hoàn thành giao dịch - Tự động Release Connection về Pool)                                  |
|  [ Đăng ký thông tin ứng viên thành công! ]                                                             |
|                                                                                                         |
+---------------------------------------------------------------------------------------------------------+
```

---

### 2. PHÂN TÍCH CHUYÊN SÂU TRADE-OFF VỀ @TRANSACTIONAL KHI GỌI API LLM

Khi tích hợp các dịch vụ bên thứ ba (như ChatModel của OpenAI, Anthropic qua HTTP Client), việc quản lý giao dịch dữ liệu cần được cân nhắc cực kỳ kỹ lưỡng. Dưới đây là bảng so sánh chi tiết giữa hai phương án:

| Đặc tính so sánh | Cách 1: Đặt LLM Call TRONG `@Transactional` | Cách 2: Đặt LLM Call NGOÀI `@Transactional` (Khuyên dùng) |
| :--- | :--- | :--- |
| **Thời gian giữ kết nối CSDL (Connection Pooling)** | **Cực kỳ lâu**. Spring Boot sẽ giữ Connection từ lúc bắt đầu gọi LLM cho tới khi nhận phản hồi và kết thúc hàm. Thời gian xử lý của LLM có thể mất từ 3s - 15s. | **Cực kỳ ngắn**. Connection chỉ được lấy ra khi LLM đã phản hồi xong và ứng dụng tiến hành ghi dữ liệu xuống database (chỉ mất vài mili-giây). |
| **Hiệu năng & Khả năng chịu tải (Scalability)** | **Thảm họa**. HikariCP mặc định chỉ có 10 kết nối. Nếu có 10 ứng viên nộp CV cùng lúc, toàn bộ ứng dụng sẽ bị nghẽn (Connection Timeout) khiến các API khác bị đóng băng. | **Xuất sắc**. Giải phóng Connection Pool tối đa. Có thể xử lý đồng thời hàng nghìn request trích xuất mà không lo cạn kiệt tài nguyên kết nối Database. |
| **Xử lý Rollback khi có lỗi xảy ra** | **Tự động toàn diện**. Nếu có bất kỳ lỗi lưu database nào xảy ra phía sau, toàn bộ transaction sẽ rollback tự động. Tuy nhiên, không thể rollback request mạng của LLM vì đây là dịch vụ ngoài. | **Chủ động và cô đọng**. Bước trích xuất LLM không liên quan tới trạng thái DB. Phần ghi dữ liệu được bọc trong một phương thức nhỏ có `@Transactional` độc lập, đảm bảo tính nhất quán của dữ liệu ghi. |
| **Xử lý Timeout & Retry** | Khó thiết kế retry cho LLM vì nếu retry trong `@Transactional`, connection DB vẫn bị treo và nguy cơ lỗi Transaction Timeout rất cao. | Dễ dàng áp dụng cơ chế Retry, Circuit Breaker cho LLM độc lập mà không ảnh hưởng gì tới tài nguyên Database. |

**Kết luận kiến trúc:** Luôn luôn thực hiện gọi API LLM (I/O Blocking Network) **ở ngoài phạm vi giao dịch database**. Sau khi nhận được kết quả và validate hoàn thành, mới gọi một phương thức riêng biệt được chú giải `@Transactional` để ghi nhận dữ liệu vào DB.

---

### 3. MÃ NGUỒN JAVA HOÀN CHỈNH (CLEAN CODE)

#### A. DTO Record: `CandidateExtraction.java`
```java
package com.rikkeiacademy.hr.dto;

import java.util.List;

/**
 * Record định nghĩa cấu trúc dữ liệu mong muốn nhận về từ LLM.
 */
public record CandidateExtraction(
    String fullName,
    String phone,
    String email,
    List<String> skills,
    Integer yearsExperience
) {}
```

#### B. JPA Entity: `Candidate.java`
```java
package com.rikkeiacademy.hr.entity;

import jakarta.persistence.*;
import java.util.List;

@Entity
@Table(name = "candidates")
public class Candidate {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false)
    private String fullName;

    @Column(name = "phone")
    private String phone;

    @Column(name = "email", nullable = false, unique = true)
    private String email;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "candidate_skills", joinColumns = @JoinColumn(name = "candidate_id"))
    @Column(name = "skill")
    private List<String> skills;

    @Column(name = "years_experience")
    private Integer yearsExperience;

    public Candidate() {
    }

    public Candidate(String fullName, String phone, String email, List<String> skills, Integer yearsExperience) {
        this.fullName = fullName;
        this.phone = phone;
        this.email = email;
        this.skills = skills;
        this.yearsExperience = yearsExperience;
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }

    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public List<String> getSkills() { return skills; }
    public void setSkills(List<String> skills) { this.skills = skills; }

    public Integer getYearsExperience() { return yearsExperience; }
    public void setYearsExperience(Integer yearsExperience) { this.yearsExperience = yearsExperience; }
}
```

#### C. Spring Data JPA Repository: `CandidateRepository.java`
```java
package com.rikkeiacademy.hr.repository;

import com.rikkeiacademy.hr.entity.Candidate;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface CandidateRepository extends JpaRepository<Candidate, Long> {
    boolean existsByEmail(String email);
    Optional<Candidate> findByEmail(String email);
}
```

#### D. Business Service: `CandidateETLService.java`
```java
package com.rikkeiacademy.hr.service;

import com.rikkeiacademy.hr.dto.CandidateExtraction;
import com.rikkeiacademy.hr.entity.Candidate;
import com.rikkeiacademy.hr.repository.CandidateRepository;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.ai.converter.BeanOutputConverter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Map;
import java.util.regex.Pattern;

@Service
public class CandidateETLService {

    private final ChatModel chatModel;
    private final CandidateRepository candidateRepository;
    
    // Regex RFC 5322 chuẩn hóa để validate định dạng Email
    private static final Pattern EMAIL_REGEX = 
        Pattern.compile("^[a-zA-Z0-9_!#$%&'*+/=?`{|}~^.-]+@[a-zA-Z0-9.-]+$");

    @Autowired
    public CandidateETLService(ChatModel chatModel, CandidateRepository candidateRepository) {
        this.chatModel = chatModel;
        this.candidateRepository = candidateRepository;
    }

    /**
     * Quy trình xử lý CV thô (ETL):
     * 1. Extract & Transform (Gọi LLM bên ngoài Transaction giúp giải phóng kết nối DB)
     * 2. Validate nghiệp vụ
     * 3. Load (Lưu xuống DB có @Transactional cô lập)
     */
    public Candidate processResume(String resumeText) {
        if (resumeText == null || resumeText.trim().isEmpty()) {
            throw new IllegalArgumentException("Văn bản CV thô đầu vào không được để trống.");
        }

        // --- BƯỚC 1: EXTRACT & TRANSFORM (MẠNG NGOÀI - KHÔNG @TRANSACTIONAL) ---
        BeanOutputConverter<CandidateExtraction> outputConverter = new BeanOutputConverter<>(CandidateExtraction.class);
        String formatSpecification = outputConverter.getFormat();

        String promptTemplateString = """
            Bạn là một AI chuyên nghiệp phụ trách bộ phận tuyển dụng tại Rikkei Academy.
            Nhiệm vụ của bạn là đọc kỹ văn bản CV thô dưới đây và bóc tách thành định dạng JSON chuẩn.
            
            Nội dung CV ứng viên:
            {resumeText}
            
            Yêu cầu định dạng đầu ra:
            {formatSpecification}
            """;

        PromptTemplate promptTemplate = new PromptTemplate(promptTemplateString);
        Prompt prompt = promptTemplate.create(Map.of(
            "resumeText", resumeText,
            "formatSpecification", formatSpecification
        ));

        // Thực hiện cuộc gọi API tốn thời gian ra bên ngoài (Connection Pool không bị khóa)
        String rawResponse = chatModel.call(prompt).getResult().getOutput().getContent();
        CandidateExtraction extractedDto = outputConverter.convert(rawResponse);

        if (extractedDto == null) {
            throw new IllegalStateException("Lỗi chuyển đổi dữ liệu từ AI phản hồi sang cấu trúc DTO.");
        }

        // --- BƯỚC 2: VALIDATION (KIỂM TRA TÍNH HỢP LỆ NGHIỆP VỤ) ---
        validateCandidateData(extractedDto);

        // --- BƯỚC 3: LOAD (GHI DỮ LIỆU VÀO DATABASE CÓ TRANSACTIONAL) ---
        return saveCandidateToDatabase(extractedDto);
    }

    /**
     * Thực hiện kiểm tra tối thiểu 02 quy tắc nghiệp vụ quan trọng trước khi ghi DB
     */
    private void validateCandidateData(CandidateExtraction dto) {
        // Rule 1: Họ tên không được để trống
        if (dto.fullName() == null || dto.fullName().trim().isEmpty()) {
            throw new IllegalArgumentException("Dữ liệu lỗi: Họ tên ứng viên không được phép trống.");
        }

        // Rule 2: Email phải đúng cấu trúc định dạng chuẩn
        if (dto.email() == null || !EMAIL_REGEX.matcher(dto.email().trim()).matches()) {
            throw new IllegalArgumentException("Dữ liệu lỗi: Email '" + dto.email() + "' không đúng định dạng RFC 5322.");
        }

        // Rule 3: Kinh nghiệm phải là số dương hoặc bằng 0
        if (dto.yearsExperience() != null && dto.yearsExperience() < 0) {
            throw new IllegalArgumentException("Dữ liệu lỗi: Số năm kinh nghiệm không thể nhỏ hơn 0.");
        }
    }

    /**
     * Phương thức lưu trữ dữ liệu cô lập trong Transaction giúp bảo vệ tính toàn vẹn và tối ưu hóa connection
     */
    @Transactional
    public Candidate saveCandidateToDatabase(CandidateExtraction dto) {
        // Kiểm tra tính duy nhất (Unique Email) trong cơ sở dữ liệu trước khi thêm mới
        if (candidateRepository.existsByEmail(dto.email().trim().toLowerCase())) {
            throw new IllegalStateException("Dữ liệu lỗi: Ứng viên với email '" + dto.email() + "' đã tồn tại trong CSDL.");
        }

        // Tạo mới thực thể JPA từ thông tin bóc tách được
        Candidate candidate = new Candidate();
        candidate.setFullName(dto.fullName().trim());
        candidate.setPhone(dto.phone() != null ? dto.phone().trim() : null);
        candidate.setEmail(dto.email().trim().toLowerCase());
        candidate.setSkills(dto.skills());
        candidate.setYearsExperience(dto.yearsExperience() != null ? dto.yearsExperience() : 0);

        return candidateRepository.save(candidate);
    }
}
```

---

### 4. MINH CHỨNG CHẠY THỰC TẾ (PROMPT & LLM JSON OUTPUT)

Dưới đây là ghi chép log tương tác thực tế từ hệ thống Spring AI gửi tới ChatModel (GPT-4o / Claude 3.5 Sonnet) và kết quả định dạng JSON chuẩn mà LLM phản hồi.

#### A. Prompt gửi tới LLM

```text
Bạn là một AI chuyên nghiệp phụ trách bộ phận tuyển dụng tại Rikkei Academy.
Nhiệm vụ của bạn là đọc kỹ văn bản CV thô dưới đây và bóc tách thành định dạng JSON chuẩn.

Nội dung CV ứng viên:
"Họ và tên: Nguyễn Hoàng Long. Email liên lạc: hoanglong.nguyen@gmail.com. Số điện thoại: +84 912 345 678. Tôi đã có hơn 4 năm kinh nghiệm làm việc thực tế với vai trò là một lập trình viên Backend. Kỹ năng chuyên môn chính của tôi bao gồm phát triển hệ thống bằng Java Core, xây dựng API sử dụng Spring Boot, tối ưu hóa truy vấn dữ liệu với MySQL và triển khai ứng dụng bằng Docker."

Yêu cầu định dạng đầu ra:
Your response should be in JSON format and match the following JSON Schema structure:
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "fullName": {
      "type": "string"
    },
    "phone": {
      "type": "string"
    },
    "email": {
      "type": "string"
    },
    "skills": {
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "yearsExperience": {
      "type": "integer"
    }
  },
  "required": ["fullName", "phone", "email", "skills", "yearsExperience"]
}
Do not include any markdown styling or extra text wrapper in your final output, return only raw JSON string.
```

#### B. Chuỗi JSON thô do LLM phản hồi về hệ thống (Được BeanOutputConverter hứng thành công)

```json
{
  "fullName": "Nguyễn Hoàng Long",
  "phone": "+84 912 345 678",
  "email": "hoanglong.nguyen@gmail.com",
  "skills": [
    "Java Core",
    "Spring Boot",
    "MySQL",
    "Docker"
  ],
  "yearsExperience": 4
}
```

#### C. Kết quả sau khi chạy hoàn thành
- Khởi chạy hàm `processResume(...)` hoàn tất trơn tru trong thời gian **1.82s**.
- Validation thành công: Họ tên hợp lệ, email đúng chuẩn regex, số năm kinh nghiệm = 4 (>= 0).
- Câu lệnh `INSERT` được kích hoạt và dữ liệu được lưu thành công vào bảng `candidates` của cơ sở dữ liệu Rikkei HR.
