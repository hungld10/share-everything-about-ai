# Skill System Guide  

## Mục lục  
- [1. Tư duy gốc: Kỹ năng khác gì với Prompt?](#1-tư-duy-gốc-kỹ-năng-khác-gì-với-prompt)  
- [2. Mô hình 5 lớp của hệ Skill](#2-mô-hình-5-lớp-của-hệ-skill)  
- [3. Tám loại kỹ năng cơ bản](#3-tám-loại-kỹ-năng-cơ-bản)  
- [4. Nguyên tắc vàng khi thiết kế Skill](#4-nguyên-tắc-vàng-khi-thiết-kế-skill)  
- [5. Personal Skill OS (Hệ điều hành cá nhân)](#5-personal-skill-os-hệ-điều-hành-cá-nhân)  
- [6. Project Skill OS (Hệ điều hành dự án)](#6-project-skill-os-hệ-điều-hành-dự-án)  
- [7. Quy trình triển khai 7 bước](#7-quy-trình-triển-khai-7-bước)  
- [8. Ví dụ mẫu SKILL.md](#8-ví-dụ-mẫu-skillmd)  
- [9. Cơ chế kiểm chứng (4C)](#9-cơ-chế-kiểm-chứng-4c)  
- [10. Tùy chỉnh framework cho cá nhân](#10-tùy-chỉnh-framework-cho-cá-nhân)  
- [11. Lộ trình triển khai 30 ngày](#11-lộ-trình-triển-khai-30-ngày)  
- [12. Công thức vận hành ngắn gọn](#12-công-thức-vận-hành-ngắn-gọn)  

## 1. Tư duy gốc: Kỹ năng khác gì với Prompt?  
Hầu hết người dùng AI hiện nay vẫn làm việc theo quy trình: **gửi yêu cầu** (prompt), **nhận kết quả**, rồi chỉnh sửa prompt và thử lại cho đến khi ưng ý. Cách làm này có thể tạm ổn cho các tác vụ một bước, đơn giản. Nhưng khi công việc trở nên phức tạp hơn – **nhiều bước**, **nhiều tiêu chí đúng/sai**, yêu cầu **tái sử dụng**, **kiểm chứng** và **phối hợp nhóm** – thì chỉ có prompt thôi không đủ. Bạn phải *dạy* cho AI biết rõ ngữ cảnh, quy trình và tiêu chuẩn của tổ chức. Như tác giả Ian Loe đã chỉ ra, “nếu prompt engineering vẫn là giới hạn cao nhất cho khả năng AI của bạn, thì bạn đã tụt lại phía sau”【[1](https://ianloe.medium.com/beyond-prompt-engineering-its-time-to-build-skills-for-ai-a4ff296fc216#:~:text=box)】. Thay vào đó, ta cần xây dựng một thư viện *skills* – mỗi skill là một file Markdown đóng gói tri thức, quy tắc và hướng dẫn quy trình cho một loại công việc cụ thể. Ví dụ, thay vì nhập lại công thức cấu trúc đề xuất mỗi lần, ta có thể tạo sẵn skill "viết proposal doanh nghiệp" chứa logic, phong cách và mẫu chuẩn. Như một bài viết trên Medium nhấn mạnh, kỹ năng (skill) là một file Markdown định nghĩa cách AI nên thực hiện một nhiệm vụ, để nó không phải “định hướng lại” mỗi lần【[2](https://ianloe.medium.com/beyond-prompt-engineering-its-time-to-build-skills-for-ai-a4ff296fc216#:~:text=The%20mechanism%20for%20doing%20this%2C,a%20specific%20class%20of%20task)】【[3](https://medium.com/@ooi_yee_fei/custom-claude-code-skill-auto-generating-updating-architecture-diagrams-with-excalidraw-431022f75a13#:~:text=I%20started%20with%20,investing%20in%20more%20complex%20solutions)】.  

**Ví dụ:** Giả sử bạn thường xuyên phải soạn proposal cho khách hàng doanh nghiệp. Với prompt thông thường, bạn sẽ phải miêu tả lại yêu cầu từ đầu mỗi lần. Nhưng với skill đã được gói sẵn, Claude đã “biết” ngay cấu trúc, ngôn ngữ và các phần cần có. Bạn chỉ cần kích hoạt skill đó và cung cấp thông tin cơ bản (tên khách hàng, mục tiêu…), còn lại AI sẽ tự làm theo quy trình đã học. Kết quả là workflow lặp lại được tự động hóa, nhất quán và tiết kiệm nhiều bước căn chỉnh thủ công. 

- Ví dụ mô tả workflow lặp lại (nên làm thành skill thay vì prompt lặp): tạo landing page, lập outline khóa học đào tạo, xây proposal khách hàng, scaffold dự án webapp, viết bài chia sẻ, v.v.  
- Cách dùng prompt truyền thống phù hợp với *tác vụ ngắn hạn*, *ít rủi ro*, *ít bước*. Với quy trình phức tạp hơn (kiểm thử code, xử lý dữ liệu, tạo báo cáo chi tiết), ta cần hẳn một **hệ thống Skill** có cấu trúc.  

Theo hướng dẫn của Claude, mỗi skill thực chất là một thư mục chứa `SKILL.md` cùng các tài nguyên liên quan【[4](https://gist.github.com/joyrexus/ff71917b4fc0a2cbc84974212da34a4a#:~:text=A%20skill%20is%20a%20folder,containing)】. Claude sẽ “tự học” skill này một lần rồi áp dụng lại mãi mà không phải giải thích lại từ đầu. Như Yee Fei viết, “Skills are markdown files that teach Claude how to perform specific tasks. No scripts, no servers, just knowledge files” (Kỹ năng là file Markdown dạy Claude làm nhiệm vụ cụ thể – không cần mã, không cần máy chủ, chỉ cần file tri thức)【[3](https://medium.com/@ooi_yee_fei/custom-claude-code-skill-auto-generating-updating-architecture-diagrams-with-excalidraw-431022f75a13#:~:text=I%20started%20with%20,investing%20in%20more%20complex%20solutions)】. Điều này giúp chuyển từ việc *prompt engineer* cá nhân sang *skill system* tổ chức: AI có “bộ nhớ” về quy trình, style và kiến thức mà không phải nghe diễn giải lại mỗi lần.

## 2. Mô hình 5 lớp của hệ Skill  

![Layered Skill System](./images/fig-01-layer.png)

*Figure 1 — Layered Skill System  
Minh hoạ kiến trúc nhiều lớp của skill, từ mục tiêu đến cải tiến liên tục.*

Một hệ Skill hoàn chỉnh gồm **5 lớp** chính, tương ứng với 5 khía cạnh khác nhau trong xây dựng và sử dụng kỹ năng:

### 2.1 **Intent Layer (Lớp Mục tiêu)** 

  ![Intent Strategy](./images/fig-04-intent.png)

  *Figure 2 — Intent & Strategic Direction  
  Thể hiện việc xác định mục tiêu và định hướng trước khi thực thi.*

Xác định rõ *nhiệm vụ* cần làm, *đầu ra* mong đợi và *tiêu chí đúng/sai* của nó. Lớp này trả lời câu hỏi “Chúng ta đang cố gắng đạt mục tiêu gì, theo tiêu chuẩn nào?”. Nếu không rõ ràng ở bước này, AI rất dễ sinh ra kết quả “có vẻ ổn” nhưng không thực sự giải quyết yêu cầu. Ví dụ các intents phổ biến: viết một proposal đào tạo cho doanh nghiệp, tạo outline khóa học cho nhóm sales, phân tích use-case AI cho phòng nhân sự, v.v. Hãy xác định rạch ròi: mục tiêu của nhiệm vụ là gì, khi nào thì được xem là hoàn thành, và giới hạn phạm vi ra sao. 

### 2.2 **Knowledge Layer (Lớp Tri thức)** 

  ![Knowledge Network](./images/fig-10-network.png)

  *Figure 3 — Knowledge Network  
  Minh hoạ cách tri thức được tổ chức thành mạng lưới liên kết.*

Cung cấp mọi **kiến thức nền tảng**, quy tắc và tài liệu cần thiết để AI có thể hành động chính xác. Lớp này bao gồm các guide về domain (chuyên ngành), quy định doanh nghiệp, best practices, style guide, API docs, template, voice branding, v.v. Ví dụ: tri thức về phong cách viết LinkedIn của bạn; framework đào tạo AI cho doanh nghiệp; mẫu proposal chuẩn; checklist đánh giá use-case AI; hay tài liệu sản phẩm, quy tắc code của dự án. Lớp này trả lời câu hỏi “AI cần biết gì để làm đúng?”. Theo Ian Loe, việc mã hóa kiến thức và tiêu chuẩn thành file Markdown cho phép AI “draw upon [that knowledge] consistently” mà không phải re-brief mỗi lần【[2](https://ianloe.medium.com/beyond-prompt-engineering-its-time-to-build-skills-for-ai-a4ff296fc216#:~:text=The%20mechanism%20for%20doing%20this%2C,a%20specific%20class%20of%20task)】. Tóm lại, hãy gom tất cả thông tin nền (đặc tả khách hàng, ngành, quy trình nội bộ, mẫu văn bản, v.v.) vào một kho kiến thức mà mỗi skill có thể truy cập.  

### 2.3 **Execution Layer (Lớp Thực thi)** 

  ![Execution Flow](./images/fig-02-flowchart.png)

  *Figure 4 — Workflow Design  
  Thể hiện việc thiết kế quy trình và các bước thực thi.*

Là phần hướng dẫn *cách* thực hiện nhiệm vụ, chi tiết từng bước và tài nguyên công cụ cần thiết. Thay vì chỉ dừng ở việc mô tả tổng quan, mỗi skill nên đi kèm **script**, **mẫu workflow**, **command**, **file template**, **code scaffold**, **automation script**, **checklist thao tác**,… Ví dụ: file script tạo cấu trúc dự án, template PRD, lệnh deploy tự động, truy vấn SQL kiểm tra data, prompt-template tạo case study, mã Python xử lý file CSV, mẫu agenda workshop, v.v. Lớp này trả lời câu hỏi “Cần làm gì và chạy như thế nào để hoàn thành từng bước của nhiệm vụ?”. Theo tư liệu của Claude, một skill thực chất là một thư mục có thể bao gồm `SKILL.md` hướng dẫn và cả các thư mục con như `scripts/` chứa code, `references/` chứa tài liệu, hoặc `assets/` chứa template (hình ảnh, file mẫu)【[5](https://abvijaykumar.medium.com/deep-dive-skill-md-part-1-2-09fc9a536996#:~:text=A%20skill%20is%20a%20directory,relevant%2C%20load%20up%2C%20and%20use)】【[4](https://gist.github.com/joyrexus/ff71917b4fc0a2cbc84974212da34a4a#:~:text=A%20skill%20is%20a%20folder,containing)】. Tóm lại, Execution Layer biến hướng dẫn thành các công cụ và bước thực thi cụ thể, giúp Claude tự động làm việc thay con người.  

  ![Code Execution](./images/fig-09-code.png)

  *Figure 5 — Execution & Automation  
  Minh hoạ giai đoạn thực thi bằng code và automation.*

### 2.4 **Verification Layer (Lớp Kiểm chứng)** 

  ![Verification](./images/fig-05-verification.png)

  *Figure 6 — Verification & Risk Awareness  
  Nhấn mạnh việc kiểm tra và xử lý các nhánh rẽ, tránh sai lệch.*

Phần quan trọng nhất để đảm bảo kết quả đầu ra thực sự đạt yêu cầu. Nhiều hệ thống AI thất bại không phải vì không sinh ra được kết quả, mà vì không **test, verify, so sánh** đầu ra với kỳ vọng. Một kỹ năng mạnh phải có cơ chế kiểm tra trước khi kết thúc: ví dụ các *test case*, ví dụ đầu ra mong đợi, checklist chống mẫu lỗi (“anti-patterns”), bước mô phỏng, đối chiếu thực tế, v.v. Ví dụ, với skill viết proposal: hãy kiểm tra xem proposal đã có đủ phần bắt buộc chưa, logic triển khai có xuyên suốt không, ngôn ngữ có đủ thuyết phục đối tượng không? Với skill code: chạy thử code, so sánh output với mong đợi, kiểm tra unit test. Lớp này trả lời: “Làm sao biết kết quả vừa tạo ra là đúng và có thể tin dùng?”. Xây dựng verification layer giúp nâng giá trị của skill từ “ra ý tưởng tốt” lên “ra kết quả đáng tin cậy”.  

### 2.5 **Evolution Layer (Lớp Phát triển)** 

  ![Roadmap](./images/fig-06-roadmap.png)

  *Figure 7 — Continuous Evolution  
  Thể hiện quá trình cải tiến liên tục của hệ thống.*

Kỹ năng không phải là tài liệu bất động. Sau mỗi lần sử dụng, khi gặp lỗi, đầu ra sai định dạng, hiểu thiếu yêu cầu, gặp edge case… chúng ta phải **rút kinh nghiệm** và cập nhật skill. Evolution Layer gồm ghi chú *gotchas*, *lessons learned*, lịch sử thay đổi (changelog), phiên bản nâng cấp, v.v. Mỗi khi Claude “vấp lỗi”, hãy lưu lại ở đây và cải tiến SKILL.md tương ứng. Nhờ đó, kỹ năng sẽ “thông minh hơn qua từng lần” và dữ liệu thực tế được tích hợp trở lại. Ví dụ: nếu Claude thường bỏ sót một phần quan trọng trong proposal, ghi thành bài học để thêm checklist tương ứng.  

Trong mô hình 5 lớp trên, Intent xác định **“cái gì”** (mục tiêu), Knowledge cung cấp **“cái nên biết”**, Execution hướng dẫn **“cách làm”**, Verification đảm bảo **“đã đúng”**, và Evolution trả lời **“sau mỗi lần học được gì”**. Mỗi lớp đều cần thiết để xây dựng một hệ skill hoàn chỉnh – chúng cho phép AI hiểu bối cảnh, dùng công cụ đúng và liên tục cải thiện thay vì chỉ nhảy từ prompt này sang prompt khác.  

## 3. Tám loại kỹ năng cơ bản  
Để tổ chức hệ thống skill có cấu trúc và dễ sử dụng, ta có thể chia skill thành 8 nhóm chính dựa theo vai trò:

- **Knowledge Skills:** Cung cấp nền tảng tri thức và khung tham khảo. Ví dụ: `company-overview-unicharm` (giới thiệu doanh nghiệp Unicharm), `brand-voice-mycompany` (phong cách brand của Nguyen Tiep), `genai-training-framework` (framework đào tạo AI), `enterprise-proposal-standards` (tiêu chuẩn proposal cho khách hàng enterprise). Nhóm này thường dùng để viết nội dung, phân tích, tư vấn chiến lược, thiết kế framework, đào tạo.  

- **Verification Skills:** Dùng để kiểm tra đầu ra của các task khác. Ví dụ: `verify-proposal-completeness` (kiểm tra proposal đầy đủ phần), `verify-training-outline-fit-nontech` (đánh giá outline đào tạo phù hợp không-tech), `verify-code-quality` (kiểm tra chất lượng code), `verify-slide-structure` (kiểm tra cấu trúc slide), `verify-agent-output-format` (đảm bảo định dạng đầu ra đúng). Đây thường là điểm khác biệt lớn giữa “AI trả lời hay” và “AI làm việc đáng tin”: các verification skill giúp tự động so sánh kết quả AI với tiêu chí mong đợi.  

- **Data Skills:** Xử lý và phân tích dữ liệu. Ví dụ: `analyze-excel-sales-report` (phân tích báo cáo doanh số Excel), `clean-customer-list` (làm sạch danh sách khách hàng), `extract-insights-from-survey` (rút insight từ khảo sát), `compare-training-needs-by-department` (so sánh nhu cầu đào tạo theo phòng ban). Những skill này chạy các query, script hoặc workflow để đọc, làm sạch, tổng hợp và trực quan dữ liệu.  

- **Automation Skills:** Thực hiện các chuỗi thao tác lặp đi lặp lại. Ví dụ: `create-proposal-folder-structure` (tạo cấu trúc thư mục proposal), `generate-client-kickoff-pack` (tạo bộ tài liệu khởi động dự án cho khách hàng), `summarize-meeting-and-create-actions` (tóm tắt cuộc họp và tạo todo list), `publish-content-multi-platform` (xuất bản nội dung lên nhiều nền tảng). Các skill này có thể gọi tới công cụ ngoài, chạy lệnh tự động (deploy, script) để hoàn thành nhiệm vụ.  

- **Scaffolding Skills:** Dùng để dựng khung ban đầu cho project hay tài liệu. Ví dụ: `scaffold-laravel-project` (khởi tạo cấu trúc project Laravel), `scaffold-training-proposal` (dựng khuôn proposal đào tạo), `scaffold-course-outline` (tạo dàn ý khóa học), `scaffold-agent-skill-folder` (khởi tạo thư mục skill cho Claude). Đây là những skill tạo ra skeleton giúp tiết kiệm bước chuẩn bị.  

- **Review Skills:** Phản biện, rà soát và cải thiện đầu ra. Ví dụ: `review-business-logic` (đánh giá logic nghiệp vụ), `review-strategy-deck` (nhận xét slide chiến lược), `review-copywriting-clarity` (soát lại tính rõ ràng của bài viết), `review-system-prompt` (đánh giá prompt system). Nhóm này cho phép tự động hóa bước “coaching” trước khi giao kết quả cuối cùng.  

- **Runbook Skills:** Hướng dẫn vận hành theo quy trình. Ví dụ: `runbook-client-onboarding` (hướng dẫn onboard khách hàng), `runbook-bug-triage` (quy trình xử lý bug), `runbook-training-delivery-day` (các bước vận hành ngày tổ chức đào tạo), `runbook-demo-preparation` (quy trình chuẩn bị demo). Những runbook này viết sẵn các bước cần thực hiện theo kịch bản.  

- **Infra/CI/Delivery Skills:** Dành cho triển khai kỹ thuật và đảm bảo chất lượng hệ thống. Ví dụ: `deploy-vercel-app` (deploy ứng dụng lên Vercel), `run-pre-release-checks` (chạy kiểm tra trước phát hành), `setup-firebase-auth` (cấu hình Firebase Auth), `ci-checklist-agent-project` (checklist CI cho project Claude). Các kỹ năng này thường chứa script và quy trình cụ thể cho môi trường devops.

Mỗi nhóm skill có vai trò riêng, giúp phân loại rõ ràng, dễ tìm kiếm và tái sử dụng. Ví dụ, nhóm *Verification Skills* chính là yếu tố quyết định sự ổn định: chúng giúp tự động kiểm tra sai sót, từ đó nâng đáng giá kết quả của AI.

## 4. Nguyên tắc vàng khi thiết kế Skill  

  ![Modular System](./images/fig-07-lego-structure.png)

  *Figure 8 — Modular Skill Architecture  
  Minh hoạ cách xây dựng hệ thống từ các module có thể lắp ghép.*

Một số nguyên tắc quan trọng cần tuân theo khi xây dựng kỹ năng:  

- **(1) Một skill = một nhiệm vụ duy nhất.** Đừng gom nhiều chức năng khác nhau vào một skill. Ví dụ sai: *“skill_viết_bài_kiểm_tra_đăng_facebook_tạo_ảnh”*. Đúng: chia thành `write-facebook-post`, `verify-facebook-post`, `generate-cover-image-brief`, `publish-social-content`… Mỗi skill càng chuyên biệt, Claude càng dễ lựa chọn và thực hiện đúng. Thực tế nhiều chuyên gia khuyên: hãy tạo các *micro-skill* nhỏ, đơn nhiệm, thay vì skill khổng lồ “đa năng”【[6](https://www.linkedin.com/posts/manikandanjeeva_ai-genai-claudeskills-activity-7422130149212499968-OXFn#:~:text=TL%3BDR%3A%20Design%20AI%20systems%20around,draft)】. Mô hình này giúp giảm phức tạp và tăng khả năng bảo trì.  

- **(2) Skill là thư mục, không chỉ file đơn lẻ.** Theo hướng dẫn của Claude, một skill gồm cả thư mục chức năng, không chỉ một prompt dài. Mỗi thư mục skill có thể chứa `SKILL.md` (bắt buộc) kèm theo thư mục `scripts/` (mã chạy được), `references/` (tài liệu tham khảo), `assets/` (file mẫu, ảnh, template)…【[4](https://gist.github.com/joyrexus/ff71917b4fc0a2cbc84974212da34a4a#:~:text=A%20skill%20is%20a%20folder,containing)】. Ví dụ minh họa đơn giản:
  ```
  skill-name/
  ├── SKILL.md              ← Hướng dẫn (file Markdown chính)
  ├── scripts/              ← Script (mã tự động)
  ├── references/           ← Tài liệu bổ sung
  └── assets/               ← Tài nguyên (template, hình ảnh…)
  ```  
  Cấu trúc này giúp giữ ngữ cảnh tốt và dễ quản lý. Không nên viết mọi thứ gộp vào một file dài; thay vào đó, mỗi phần (hướng dẫn, ví dụ, code, checklist…) nên có chỗ riêng.  

- **(3) Luôn có Verification đi kèm.** Nếu thiếu bước kiểm chứng, skill chỉ là “máy phát sinh” kết quả mà thôi. Mỗi skill cần định nghĩa rõ *tiêu chí đạt*, *test case*, *checklist*, *ví dụ mẫu tốt/xấu*, v.v., để tự động kiểm tra xem đầu ra có phù hợp hay không. Ví dụ, bạn nên xây skill kiểm tra (verify) ngay sau skill tạo proposal: check xem đã có đủ phần mục tiêu, phương pháp, kết quả dự kiến chưa. Ngoài ra, hãy ghi sẵn các lỗi thường gặp (gotchas) và cách sửa tương ứng trong skill. Thiết kế như vậy giúp AI tự động **kiểm thử** kết quả trước khi “giao sản phẩm” cuối cùng, nâng mức độ tin cậy lên nhiều.  

- **(4) Cấu trúc hướng dẫn ưu tiên hơn điều khiển cứng từng bước.** Tránh liệt kê quá chi tiết kiểu “Bước 1 làm này, bước 2 làm kia, bước 3… không được lệch, v.v.” Cách làm quá kiểm soát sẽ khiến agent dễ “ngộp lệnh” và không xử lý được biến thể. Thay vào đó, hãy xác định rõ mục tiêu chung, cung cấp ngữ cảnh đầy đủ và các công cụ phù hợp, rồi để agent tự biện luận chi tiết. Tài liệu của Claude gợi ý rằng cần “thiết lập mức độ tự do phù hợp” (appropriate degrees of freedom) – nghĩa là chọn mức hướng dẫn chung hay chi tiết tùy vào độ phức tạp của nhiệm vụ【[7](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices#:~:text=Set%20appropriate%20degrees%20of%20freedom)】. Khi nhiều cách giải hợp lý, chỉ nên liệt kê bước khung, để Claude tự linh hoạt xử lý. Nói cách khác, *đừng cố điều khiển từng “cử động” của AI*, mà hãy tạo một môi trường hợp lý để nó có thể tự thích ứng và hoàn thành công việc.  

Tóm lại, mỗi skill cần được thiết kế gọn gàng, mô-đun và có khả năng kiểm soát chất lượng: **đơn nhiệm, cấu trúc rõ ràng, có cơ chế test, và đủ linh hoạt để xử lý các tình huống thực tế**. Việc coi skill như “code” (được version, review) giúp duy trì chất lượng và dễ phát triển hơn về lâu dài【[6](https://www.linkedin.com/posts/manikandanjeeva_ai-genai-claudeskills-activity-7422130149212499968-OXFn#:~:text=TL%3BDR%3A%20Design%20AI%20systems%20around,draft)】【[7](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices#:~:text=Set%20appropriate%20degrees%20of%20freedom)】.  

## 5. Personal Skill OS (Hệ điều hành cá nhân)  

  ![Skill Planning](./images/fig-03-pinboard.png)

  *Figure 9 — Skill System Planning  
  Thể hiện việc tổ chức và liên kết các skill thành hệ thống.*

Đối với nhu cầu cá nhân, ta có thể xây dựng một thư viện kỹ năng riêng, coi đó như “hệ điều hành” cho công việc hàng ngày. Một ví dụ chia nhóm skill theo 6 cụm chức năng:  

- **Thinking Skills (Tư duy):** Giúp phân tích, viết lách chiến lược và phát triển ý tưởng. Ví dụ: `think-strategically`, `break-down-complex-problem`, `synthesize-long-report`, `write-in-my-voice`, `build-framework-from-ideas`. Nhóm này rất hữu ích khi bạn cần viết bài chia sẻ AI, phân tích xu hướng công nghệ, tạo framework từ ý tưởng, hay viết theo giọng cá nhân.  

- **Content Skills (Nội dung):** Tạo và chỉnh sửa nội dung. Ví dụ: `write-facebook-post`, `write-linkedin-article`, `write-training-proposal-summary`, `write-seo-article`, `create-video-script-shortform`. Dùng cho công việc sáng tạo nội dung đa dạng: đăng bài, viết blog, lên kế hoạch bài thuyết trình, kịch bản video…  

- **Teaching Skills (Giảng dạy):** Hỗ trợ thiết kế chương trình đào tạo và tài liệu học. Ví dụ: `design-nontech-ai-training`, `build-case-study-by-department`, `create-hands-on-exercises`, `adapt-content-for-executives`, `generate-training-assessment`. Nhóm này bao gồm việc tạo khung chương trình, phân nhóm trường hợp theo phòng ban, bài tập thực hành, điều chỉnh nội dung cho đối tượng non-tech, v.v.  

- **Consulting Skills (Tư vấn):** Dùng trong các dự án tư vấn, khảo sát nhu cầu, phân tích use-case. Ví dụ: `diagnose-enterprise-ai-readiness`, `build-ai-usecase-map`, `design-interview-question-bank`, `write-prd-for-client`, `design-ai-roadmap`. Giúp bạn nhanh chóng đánh giá mức độ sẵn sàng về AI của doanh nghiệp, lập bản đồ các use-case AI, soạn bộ câu hỏi phỏng vấn, viết Product Requirement Doc (PRD) cho khách, xây dựng lộ trình AI...  

- **Build Skills (Xây dựng):** Dành cho prototyping, code và tự động hóa. Ví dụ: `scaffold-agent-project`, `build-laravel-module`, `create-database-schema`, `create-n8n-workflow-spec`, `setup-rag-structure`. Bao gồm khởi tạo cấu trúc project (ví dụ tạo project Agent, Laravel), xây module code, tạo sơ đồ database, thiết kế luồng RAG (retrieval-augmented generation)…  

- **Operating Skills (Vận hành):** Hỗ trợ công việc cá nhân và nhóm hàng ngày. Ví dụ: `meeting-note-to-actions` (chuyển ghi chú họp thành kế hoạch), `weekly-review-and-planning` (tổng kết tuần và lập kế hoạch mới), `create-client-delivery-checklist` (checklist bàn giao khách hàng), `update-knowledge-from-lessons` (cập nhật kiến thức từ các lessons learned), `postmortem-after-project` (phân tích sau dự án). Những skill này giúp tổ chức công việc, lên kế hoạch và rút kinh nghiệm.  

Mỗi cụm trên giúp bạn tổ chức thư viện kỹ năng rõ ràng theo loại công việc: từ tư duy-chiến lược đến content, đào tạo, tư vấn, kỹ thuật, vận hành. Ví dụ, nếu bạn thường viết bài chia sẻ và phân tích, hãy đầu tư vào các skill nhóm *Thinking Skills* và *Content Skills*. Nếu bạn hay thiết kế khóa đào tạo, tập trung vào *Teaching Skills*. Nhóm tư vấn và xây dựng thì hỗ trợ khi bạn làm dự án cho khách hoặc dev prototype. Bằng cách phân loại, bạn có thể phát triển toàn diện các kỹ năng AI phù hợp với nhu cầu cá nhân.  

## 6. Project Skill OS (Hệ điều hành cho dự án)  
Khi triển khai cho từng dự án cụ thể, bạn có thể coi mỗi dự án như một “mini-OS” có cấu trúc thư mục rõ ràng. Một ví dụ cấu trúc như sau:  

```
project-name/  
├── context/                   ← thông tin bối cảnh dự án (business goals, stakeholders, scope…)  
├── skills/                    ← chứa các thư mục skill con (phân theo loại như knowledge/, verification/, automation/,…)  
├── templates/                 ← mẫu tài liệu, mẫu code, ví dụ…  
├── data/                      ← dữ liệu mẫu, schema, mapping liên quan  
├── scripts/                   ← mã nguồn (script) phục vụ dự án  
├── outputs/                   ← kết quả đã tạo (báo cáo, code, báo cáo…)  
├── logs/                      ← nhật ký quá trình chạy (log của agent, output lịch sử…)  
└── lessons/                   ← bài học, ghi chú sau khi dùng (gotchas, changelog, failure cases…)  
```  

Trong đó:  
- **context/** chứa tài liệu giúp Claude hiểu bài toán (mục tiêu kinh doanh, stakeholders, phạm vi, thuật ngữ dự án…).  
- **skills/** chứa các skill chức năng của dự án, chia theo thư mục con (knowledge, verification, automation, v.v.).  
- **templates/**, **data/**, **scripts/** chứa lần lượt các mẫu tài liệu hoặc code, dữ liệu mẫu, và script thực thi.  
- **outputs/** lưu kết quả đã tạo (báo cáo, mã, phân tích…).  
- **logs/** ghi lại các nhật ký quá trình thực thi của agent.  
- **lessons/** lưu lại kinh nghiệm, lỗi thường gặp để cải thiện.  

Cấu trúc này giúp mỗi dự án có “khung xương” logic: mọi thứ từ bối cảnh đến công cụ thi hành và bài học đều được tổ chức rõ ràng. Khi Claude làm việc trong project, nó có thể dễ dàng truy cập vào thư mục skills/ để chọn skill phù hợp, dùng templates và scripts sẵn có, và cuối cùng ghi output vào thư mục outputs/.  

## 7. Quy trình triển khai 7 bước 

  ![Process Flow](./images/fig-02-flowchart.png)

  *Figure 11 — Process Thinking  
  Thể hiện việc chuẩn hoá quy trình thành các bước rõ ràng.*

Để thực chiến, bạn có thể áp dụng quy trình 7 bước sau:  

1. **Xác định các tác vụ lặp lại giá trị cao:** 

    ![Raw Input](./images/fig-08-lego-chaos.png)

    *Figure 10 — From Chaos to System  
    Minh hoạ trạng thái hỗn loạn trước khi được tổ chức thành skill system.*

    Liệt kê 5–10 công việc bạn làm thường xuyên nhất – đặc biệt là việc dễ sai, tốn thời gian hoặc có format cố định. Ví dụ: viết proposal, tạo outline đào tạo, phân tích khảo sát nhu cầu, review slide, tạo prompt hệ thống, scaffold dự án webapp, viết bài AI chia sẻ, v.v. Đây là những nhiệm vụ tốt nhất để biến thành skill.  

2. **Tách tác vụ thành đơn vị độc lập:** 

    Chia nhỏ từng tác vụ lớn thành các bước hay công việc nhỏ rõ ràng. Ví dụ thay vì một skill “làm proposal” chung chung, có thể tách thành `analyze-client-need`, `generate-proposal-outline`, `write-proposal-executive-summary`, `build-pricing-options`, `verify-proposal-completeness`,... Mỗi unit cụ thể, dễ tái sử dụng cho nhiều kịch bản hơn.  

3. **Đóng gói mỗi skill dưới dạng thư mục:** 

    Mỗi unit sau khi tách hãy đóng gói thành thư mục riêng. Tối thiểu mỗi skill cần có:  
    - `SKILL.md` (hướng dẫn chính)  
    - thư mục `examples/` (ví dụ đầu ra mẫu)  
    - thư mục `templates/` (các file mẫu, code mẫu)  
    - thư mục `verification/` (định nghĩa checklist, test case)  
    - `changelog.md` (lịch sử thay đổi)  
    Nếu skill liên quan kỹ thuật, có thể thêm `scripts/` chứa code thực thi và `test-data/` chứa dữ liệu thử.  

4. **Viết SKILL.md theo cấu trúc chuẩn:** 

    Trong `SKILL.md`, nêu rõ Mục đích, Khi nào dùng, Đầu vào cần, Đầu ra mong đợi, Quy trình thực hiện (các bước chính), Tiêu chí chất lượng, Verification (làm sao kiểm tra), Edge cases, Ví dụ minh họa, Changelog. Mỗi mục nên ngắn gọn và rõ ràng. (Xem phần Ví dụ dưới đây để tham khảo định dạng.) 

5. **Đính kèm template, ví dụ và checklist:** 

    Bổ sung cho mỗi skill các tài nguyên cần thiết ở trên (mẫu văn bản, mẫu code, script, ví dụ đầu ra tốt/xấu, checklist). Việc này biến skill từ “hướng dẫn” thành “mô-đun dùng được” ngay lập tức. Ví dụ: với skill proposal, hãy gắn kèm `templates/proposal-outline.md`, `examples/sample-proposal.docx`, `verification/checklist.md`… để Claude có đủ thông tin.  

6. **Sử dụng thực tế trên vài trường hợp:** 

    Dùng skill đã tạo vào 5–10 bài toán cụ thể. Ghi lại: skill nào giúp tiết kiệm thời gian nhất, skill nào ra kết quả còn nhiều lỗi, lỗi thường gặp xuất hiện ở đâu (ví dụ AI hay bỏ sót mục nào, gặp data format không xử lý được…). Theo dõi hiệu suất và ghi lại mọi thiếu sót.  

7. **Cập nhật và học hỏi (Create gotchas & Evolve):** 

    Sau mỗi lần sử dụng, nếu skill xảy ra lỗi hoặc output không chính xác, hãy cập nhật `SKILL.md` ngay. Thêm mục lessons learned, gotchas (những lưu ý dễ sai), và nâng version trong changelog. Như vậy, hệ skill sẽ ngày càng hoàn thiện dựa trên dữ liệu thực tế.  

Tuân thủ quy trình trên sẽ giúp bạn biến những công việc thường xuyên thành các module tự động hóa chuẩn mực. Dần dà bạn sẽ xây dựng được một thư viện skill hữu dụng, giảm thiểu sai sót và tăng tốc công việc.  

## 8. Ví dụ mẫu `SKILL.md`  
Dưới đây là ví dụ minh họa cấu trúc một file `SKILL.md` mẫu (mỗi đoạn nội dung bạn viết có thể tương ứng với phần trong quy trình ở trên). Chú ý rằng phần frontmatter (sau `# SKILL:`) là một thẻ Markdown để phân biệt:  

```markdown
# SKILL: write-enterprise-proposal

## Purpose
Tạo proposal đào tạo/tư vấn chuẩn enterprise, logic rõ ràng, hướng đến lãnh đạo doanh nghiệp.

## Use When
- Cần viết proposal cho khách hàng doanh nghiệp.
- Cần chuẩn hóa cấu trúc đề xuất.
- Cần giữ phong cách chuyên nghiệp, chiến lược.

## Required Inputs
- Tên khách hàng.
- Bối cảnh / nhu cầu của khách.
- Đối tượng học viên.
- Mục tiêu chương trình.
- Thời lượng dự kiến.
- Yêu cầu đặc thù (nếu có).

## Expected Output
- Một proposal có cấu trúc hoàn chỉnh.
- Ngôn ngữ rõ ràng, thuyết phục, không quá học thuật.
- Theo logic: bối cảnh → vấn đề → mục tiêu → chương trình → phương pháp → kết quả kỳ vọng.

## Execution Approach
1. Phân tích bối cảnh và nhu cầu chính của doanh nghiệp.
2. Nhóm nhu cầu thành 3–5 mục tiêu trọng tâm.
3. Thiết kế chương trình đào tạo phù hợp với từng nhóm đối tượng.
4. Viết proposal theo cấu trúc chuẩn.
5. Kiểm tra lại sự rõ ràng, tính khả thi và sức thuyết phục của đề xuất.

## Quality Criteria
- Nội dung không lan man, đúng trọng tâm.
- Ngôn ngữ phù hợp ngành, chuyên nghiệp nhưng dễ hiểu.
- Phù hợp với đối tượng non-tech (nếu có yêu cầu).
- Đề xuất có kết quả/mục tiêu cụ thể, không chung chung.

## Verification
- Kiểm tra xem proposal đã có tất cả các phần bắt buộc chưa.
- Đầu ra có logic triển khai (mục tiêu → kết quả) không.
- Ngôn ngữ có phù hợp đối tượng và phong cách (ví dụ so sánh với tone mẫu) không.
- So sánh với ví dụ mẫu (nếu có) để đảm bảo tính nhất quán.
- Đã bám sát đầu vào (nhu cầu khách hàng) chưa, chưa có nội dung nào lạc đề?

## Edge Cases
- Nhu cầu khách hàng mô tả mơ hồ hoặc rất rộng.
- Yêu cầu vượt quá thời lượng cho phép.
- Đối tượng học viên quá đa dạng (non-tech pha trộn với kỹ thuật).
- Mong muốn “thực chiến” nhưng chưa có data thực tế để dẫn chứng.

## References
- `templates/proposal-outline.md` (mẫu dàn ý)
- `examples/sample-proposal-b2b.md` (ví dụ đề xuất)
- `verification/checklist.md` (danh sách kiểm tra chuẩn)

## Changelog
- v1.0: Khởi tạo cơ bản, đã có outline chuẩn.
- v1.1: Thêm mục tiêu chi tiết cho từng phòng ban.
- v1.2: Cập nhật checklist kiểm tra, bổ sung ví dụ minh họa.
```

*Ghi chú:* Ví dụ trên chỉ mang tính minh họa. Trong thực tế, bạn tùy chỉnh nội dung và cấu trúc cho phù hợp với từng nhiệm vụ.  

## 9. Cơ chế kiểm chứng (4C)  
Một framework hữu ích để thiết kế bước kiểm chứng cho mọi skill là **4C**: *Correctness, Completeness, Context-fit, Consequence*.  

- **Correctness (Đúng đắn):** Kết quả có chính xác không? Đúng logic chưa, đúng dữ kiện chưa, đúng quy trình/định dạng chưa? Ví dụ: code có chạy đúng, proposal có tuân thủ cấu trúc yêu cầu.  
- **Completeness (Đầy đủ):** Kết quả có thiếu phần nào không? Tất cả phần bắt buộc đã có chưa, có bỏ sót bước quan trọng nào không?  
- **Context-fit (Phù hợp ngữ cảnh):** Kết quả có phù hợp với bối cảnh không? Nội dung đúng ngành chưa, đúng đối tượng (non-tech/ kỹ thuật) chưa, có bám sát mục tiêu đề bài không?  
- **Consequence (Hậu quả):** Nếu kết quả này được sử dụng thực tế, có rủi ro gì? Ví dụ: nếu gửi proposal này cho khách, có điều gì sẽ bị hiểu sai hay gây phiền hà? Nếu chạy script thật, có nguy cơ lỗi hệ thống?  

Mỗi skill nên tự trả lời được 4 câu hỏi trên. Cuối mỗi phần output, hãy thêm câu hỏi: *“Nếu kết quả này được đưa vào thực tế ngay lập tức, rủi ro lớn nhất là gì?”* để nâng cao độ tin cậy. Bằng cách này, ta biến đầu ra “có vẻ ổn” thành “đủ tin cậy để dùng”, đảm bảo khách quan và giảm thiểu sai sót.  

## 10. Tùy chỉnh framework cho cá nhân  
Dựa trên định hướng cá nhân, bạn có thể tùy chỉnh bộ skill của mình thành 4 trục chính:  

- **Trục 1: Kỹ năng tư duy & viết chiến lược AI.** Dành cho phân tích, viết bài chia sẻ, nghiên cứu. Ví dụ các skill: `translate-research-into-business-insight` (biên dịch nghiên cứu thành insight), `write-ai-article-in-my-voice` (viết bài AI theo giọng tôi), `build-framework-from-research`, `summarize-report-with-insights`, `write-facebook-short-version` (viết bản ngắn cho Facebook).  

- **Trục 2: Kỹ năng đào tạo doanh nghiệp.** Tạo nội dung đào tạo và case-study. Ví dụ: `design-enterprise-ai-training`, `adapt-content-for-nontech`, `create-department-usecases`, `write-training-proposal`, `verify-training-flow`, `build-executive-version`. Nhóm này hữu ích khi bạn thiết kế khóa học, đề xuất chương trình, tổ chức training.  

- **Trục 3: Kỹ năng phát triển sản phẩm/agent/project.** Dùng cho phần kỹ thuật và phát triển agent. Ví dụ: `scaffold-agent-skill-system`, `design-system-prompt`, `build-rag-folder-structure`, `define-agent-tools-and-constraints`, `verify-agent-behavior`, `document-lessons-from-failures`. Giúp bạn xây dựng khung dự án Claude, thiết kế hệ thống prompt, cấu trúc RAG, kiểm thử agent, và ghi nhận các bài học.  

- **Trục 4: Kỹ năng vận hành kinh doanh & triển khai.** Hỗ trợ thu thập yêu cầu, họp hành và chuyển tải. Ví dụ: `client-brief-to-prd` (từ khách hàng đến PRD), `meeting-notes-to-action-plan`, `diagnose-ai-readiness`, `generate-interview-question-set`, `create-proposal-followup-email`. Nhóm này giúp bạn chuẩn hóa cách lấy yêu cầu, soạn mail theo dõi, đánh giá readiness…  

Bạn có thể tự tạo thêm các skill phù hợp với nhu cầu riêng. Ví dụ, nếu bạn thường xuyên làm báo cáo dự án, hãy viết skill `analyze-project-status`. Nếu hay làm slide cho training, tạo skill `scaffold-presentation`. Quan trọng là hướng tới mục tiêu cụ thể của bạn.  

## 11. Lộ trình triển khai 30 ngày  
Dưới đây là gợi ý chia tiến trình xây skill thành 4 giai đoạn trong 30 ngày:  

- **Giai đoạn 1 (Ngày 1–7):** Gom lại 10 tác vụ lặp lại nhất mà bạn hay làm (xác định từ các gợi ý trên). Đánh giá những task vừa lặp lại, vừa giá trị để ưu tiên.  

- **Giai đoạn 2 (Ngày 8–14):** Chuẩn hóa 5 skill đầu tiên. Ưu tiên làm mẫu:  
  - 2 skill viết nội dung (Content Skills)  
  - 2 skill đào tạo/tư vấn (Teaching/Consulting)  
  - 1 skill verification (kiểm tra)  
  Viết xong `SKILL.md` cơ bản cho từng skill này, đặt vào thư mục, thêm templates và ví dụ đơn giản.  

- **Giai đoạn 3 (Ngày 15–20):** Hoàn thiện chi tiết cho 5 skill đó. Đính kèm đầy đủ mẫu tài liệu, ví dụ đầu ra (example), checklist kiểm tra, kịch bản thực thi. Kiểm thử vài lần để chắc skill chạy ổn.  

- **Giai đoạn 4 (Ngày 21–27):** Thực dùng các skill vào 5–10 bài toán thực tế. Quan sát skill nào giúp tiết kiệm thời gian nhiều nhất, skill nào ra output chưa ổn, lỗi lặp lại ở đâu. Ghi lại các lỗi/sai sót gặp phải.  

- **Giai đoạn 5 (Ngày 28–30):** Cập nhật lessons learned: với mỗi lỗi gặp, thêm ghi chú vào phần lessons hoặc chỉnh `SKILL.md`. Tạo danh sách “gotchas” và đóng nhãn phiên bản mới (version). Xem lại toàn bộ skill, đảm bảo mỗi skill đều có phần verification và cập nhật. 

Tuân thủ roadmap trên sẽ giúp bạn bắt đầu từ từ, tích lũy skill và dần hoàn thiện hệ thống một cách bền vững.  

## 12. Công thức vận hành ngắn gọn  

  ![System Roadmap](./images/fig-06-roadmap.png)

  *Figure 12 — System Evolution Loop  
  Minh hoạ vòng lặp phát triển liên tục của skill system.*

Một công thức để nhớ hệ quy trình vận hành skill:  
```
SCOPE → SKILL → EXECUTE → VERIFY → EVOLVE
```  
- **Scope (Xác định bài toán):** Xác định rõ *mục tiêu*, đầu ra mong đợi và tiêu chí thành công.  
- **Skill (Chọn/thiết kế module):** Chọn skill phù hợp hoặc tạo mới, dựa trên những nhiệm vụ đã chuẩn hóa. Kỹ năng này đóng gói tri thức và quy trình.  
- **Execute (Thực thi):** Yêu cầu Claude thực thi skill đã chọn, sử dụng ngữ cảnh và workflow chuẩn.  
- **Verify (Kiểm tra):** Đối chiếu kết quả với checklist/test đã định nghĩa; sửa đổi hoặc lặp lại nếu chưa đạt.  
- **Evolve (Cập nhật):** Rút kinh nghiệm từ lỗi và edge case; cập nhật vào SKILL.md để lần sau AI thông minh hơn.  

Đây là một vòng lặp liên tục tạo ra năng lực tích lũy. Mỗi lần lặp lại – từ xác định scope đến chỉnh sửa skill – đều giúp cải thiện hệ thống tổng thể. Hệ Skill không chỉ giúp bạn tăng tốc công việc một cách nhất quán, mà còn cho phép mở rộng năng lực cá nhân và tổ chức theo thời gian.  

## **Nguồn tham khảo:** 

Những ý tưởng và khái niệm trong tài liệu trên được tổng hợp và biên dịch từ các hướng dẫn chính thức của Claude (Anthropic) và các bài viết chuyên sâu: chẳng hạn hướng dẫn xây Skill của Anthropic【[4](https://gist.github.com/joyrexus/ff71917b4fc0a2cbc84974212da34a4a#:~:text=A%20skill%20is%20a%20folder,containing)】【[5](https://abvijaykumar.medium.com/deep-dive-skill-md-part-1-2-09fc9a536996#:~:text=A%20skill%20is%20a%20directory,relevant%2C%20load%20up%2C%20and%20use)】, bài viết “Beyond Prompt Engineering” của Ian Loe【[1](https://ianloe.medium.com/beyond-prompt-engineering-its-time-to-build-skills-for-ai-a4ff296fc216#:~:text=box)】【[2](https://ianloe.medium.com/beyond-prompt-engineering-its-time-to-build-skills-for-ai-a4ff296fc216#:~:text=The%20mechanism%20for%20doing%20this%2C,a%20specific%20class%20of%20task)】, và các kinh nghiệm từ cộng đồng xây dựng Claude Skills【[6](https://www.linkedin.com/posts/manikandanjeeva_ai-genai-claudeskills-activity-7422130149212499968-OXFn#:~:text=TL%3BDR%3A%20Design%20AI%20systems%20around,draft)】【[3](https://medium.com/@ooi_yee_fei/custom-claude-code-skill-auto-generating-updating-architecture-diagrams-with-excalidraw-431022f75a13#:~:text=I%20started%20with%20,investing%20in%20more%20complex%20solutions)】. Các nguồn này nhấn mạnh sự khác biệt giữa prompt engineering và skill system, cũng như cách tổ chức hệ thống kỹ năng để AI hoạt động hiệu quả và tin cậy hơn.

 
[1] [2] Beyond Prompt Engineering: It’s Time to Build Skills for AI | by Ian Loe | Mar, 2026 | Medium
https://ianloe.medium.com/beyond-prompt-engineering-its-time-to-build-skills-for-ai-a4ff296fc216

[3] Custom Claude Code Skill: Auto-Generating / Updating Architecture Diagrams with Excalidraw | by Yee Fei | Medium
https://medium.com/@ooi_yee_fei/custom-claude-code-skill-auto-generating-updating-architecture-diagrams-with-excalidraw-431022f75a13

[4] Complete guide to building Skills for Claude — covers fundamentals, planning, testing, distribution, patterns, and YAML frontmatter reference (converted from Anthropic's official PDF) · GitHub
https://gist.github.com/joyrexus/ff71917b4fc0a2cbc84974212da34a4a

[5] Deep Dive SKILL.md (Part 1/2). Lately I have been able to do most of… | by A B Vijay Kumar | Mar, 2026 | Medium
https://abvijaykumar.medium.com/deep-dive-skill-md-part-1-2-09fc9a536996

[6] Shifting from Heavy Prompts to Skills Architecture | Manikandan Jeeva posted on the topic | LinkedIn
https://www.linkedin.com/posts/manikandanjeeva_ai-genai-claudeskills-activity-7422130149212499968-OXFn

[7] Skill authoring best practices - Claude API Docs
https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
