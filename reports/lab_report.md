# Bao Cao Thuc Hanh & Thuyet Minh Ky Thuat - Lab 19: GraphRAG vs Flat RAG

**Hoc vien:** Nguyen Minh Huy  
**Khoa hoc:** AICB-K34 - Track 3: GraphRAG  
**Ngay thuc hien:** 19/08/2026  

---

## PHAN 1: THUYET MINH KY THUAT & PHAN TICH CA LOI

### 1. Coreference Resolution

- **Vi du tu du lieu:** pipeline local chay tren 5,000 dong HackerNoon, lay 120 chunks cho coreference/extraction. Mot dang cau kho la cac description ngan kieu "the company announced...", "it launched...", trong do chunk chi co headline/description ngan va khong co ngu canh truoc/sau day du.
- **Hien tuong:** voi conservative coreference, neu antecedent khong ro trong cung chunk thi he thong nen giu nguyen dai tu/generic mention thay vi gan bua vao cong ty gan nhat.
- **Hau qua doi voi graph:** false coreference co the tao false edge, vi du gan nham quan he `DEVELOPED` hoac `ACQUIRED` cho cong ty duoc nhac gan do nhung khong phai chu the that.

### 2. Entity Resolution Threshold & Lexical Guard

- **Nguong cosine similarity:** `threshold = 0.90` cho vector matching trong `build_resolution_map()`.
- **Co che guard:** sau khi vector similarity de xuat cap merge, `merge_guard()` tiep tuc dung lexical normalization va `SequenceMatcher` ratio de chan false merge.
- **Cap canh bao:** cac ten cong ty/san pham ngan hoac co hau to giong nhau, vi du `First Orion Hiya` voi cac entity telecom khac, co the gan ve cung mien ngu nghia nhung khong duoc merge neu lexical guard khong dat.
- **Ly do:** trong GraphRAG production, false merge nguy hiem hon missing merge, vi no lam cac edge cua hai thuc the khac nhau bi tron vao mot node canonical.

### 3. Do Thi & Super-node Mitigation

Ket qua Neo4j sau lan chay local:

| Hang | Ten thuc the | Type | Degree |
|---|---|---|---|
| 1 | payroll technology | Technology | 1 |
| 2 | First Orion Hiya | Company | 1 |
| 3 | Fidelity National Information Services | Company | 1 |

Tong quan graph:

- Nodes: `10`
- Edges: `5`
- `invalid_provenance_edges = 0`

Trong sample local nay chua xuat hien super-node degree > 100. Tuy vay, policy trong `retrieve_graph_context()` van dung `SUPER_NODE_DEGREE = 100`, `SUPER_NODE_EDGE_CAP = 50`, va `GLOBAL_EDGE_CAP = 250`.

- **Uu diem cua temporal cap:** tranh bung no context/token khi gap node lon nhu Google, Microsoft; uu tien canh moi hon de phu hop news domain.
- **Rui ro:** neu cau hoi hoi su kien cu, cap theo `published_date DESC` co the cat mat canh lich su quan trong.

### 4. So Sanh Thuc Nghiem Flat RAG vs GraphRAG

Luu y: Groq cham daily token limit o Phan 4, nen notebook da dung fallback heuristic judge de xuat CSV. Rationale trong CSV ghi ro dieu nay. Phan extraction/generation truoc do van da chay local tren subset.

| Tieu chi | Flat RAG | GraphRAG | Delta | Nhan xet |
|---|---:|---:|---:|---|
| Comprehensiveness factoid | 1.0 | 5.0 | +4.0 | GraphRAG tim dung edge co provenance. |
| Faithfulness factoid | 4.0 | 5.0 | +1.0 | Graph context bam vao evidence hon. |
| Multi-hop reasoning factoid | 1.0 | 4.0 | +3.0 | Flat RAG chi tra chunks, GraphRAG co relation line. |
| Comprehensiveness multi-hop | 1.5 | 3.0 | +1.5 | GraphRAG tot hon nhung graph nho nen chua manh. |
| Comprehensiveness cross-doc | 1.5 | 3.0 | +1.5 | GraphRAG co loi the khi entity/relation co trong graph. |

#### Ca loi Flat RAG that bai, GraphRAG thanh cong

- **Question ID:** `G01`
- **Cau hoi:** relationship gi ket noi `First Orion Hiya` va `Neustar TNS`.
- **Flat RAG loi:** vector search retrieve dung chunk lien quan nhung answer fallback chi gom chunk snippets, khong linearize quan he.
- **GraphRAG thanh cong:** graph context co dong `First Orion Hiya [Company] -PARTNERED_WITH-> Neustar TNS [Company]`, kem `date=2022-12-28`, `chunk=47d50bb1c2bd5e9f2a45::c0000`, va evidence.

#### Ca loi GraphRAG kho khan

- **Question ID:** `G04`/`G05`
- **Nguyen nhan:** graph local chi co 5 edges do gioi han rate limit va `EXTRACTION_MAX_CHUNKS = 120`, nen nhieu cau hoi multi-hop/cross-doc khong co du chain trong graph.
- **Khac phuc:** tang extraction chunks khi quota LLM reset, them caching/checkpoint cho extraction, va bo sung community/global search neu graph lon hon.

### 5. Trade-offs, Agent Control & Scale 350MB

- **Quality vs cost vs latency:** Flat RAG re va de build hon; GraphRAG ton chi phi extraction/Neo4j/entity resolution nhung cho truy van co cau truc va provenance tot hon.
- **Quyet dinh tu choi AI Coding Agent:** khong chap nhan hard-code API key vao notebook; khong dung pairwise O(N^2) entity matching tren toan bo dataset; khong tiep tuc goi Groq khi da cham daily token limit.
- **Scale 350MB:** bottleneck dau tien la LLM extraction va entity resolution. Giai phap: batch queue, retry/backoff, checkpoint tung batch, HNSW/FAISS blocking cho entity resolution, va partition/community detection de giam traversal fan-out.

---

## PHAN 2: SUY NGAM & KE HOACH DO AN

### 1. Mapping Bai Giang Vao Code

| Khai niem | Module | Ham / Khoi code | Quan sat |
|---|---|---|---|
| Conservative Coreference | Module 1 | `resolve_coref_batch()`, `run_coref()` | Nen uu tien precision, log unresolved mentions. |
| Schema & Allowlist Guard | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Giam hallucinated labels/relations. |
| Bulk Cypher Ingestion | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Dung `UNWIND`, khong insert tung row. |
| Entity Resolution | Module 3 | `build_resolution_map()`, `UF`, `merge_guard()` | Vector similarity can lexical guard. |
| Super-node Degree Cap | Module 4 | `retrieve_graph_context()` | Cap degree va global edge de tranh no context. |
| Evaluation | Module 5 | `run_evaluation()`, `judge_answer()` | Can LLM quota; khi het quota phai ghi ro fallback. |

### 2. Debugging & Bai Hoc

- **Loi kho nhat:** Groq model trong `.env` ban dau (`llama-3.3-70b-versatile`) khong kha dung voi account, sau do `openai/gpt-oss-20b` lai cham daily token limit o Phan 4.
- **Cach xu ly:** doi model sang model account co quyen, giam `EXTRACTION_MAX_CHUNKS` xuong 120 de chay local, them fallback cho evaluation de van xuat CSV va ghi ro han che.
- **Loi Neo4j:** Aura connection bi stale khi batch insert sau thoi gian LLM lau. Da them retry/reconnect trong `run_cypher()`.

### 3. Ke Hoach Ap Dung Vao Do An

- **Ten do an:** Tro ly hoi dap tai lieu doanh nghiep/noi bo.
- **Co can GraphRAG khong:** can Hybrid RAG neu cau hoi yeu cau suy luan giua nhieu tai lieu, nguoi, phong ban, san pham, deadline. Neu chi FAQ don gian thi Flat RAG du.
- **Nodes du kien:** `Document`, `Person`, `Team`, `Project`, `Product`, `Policy`.
- **Relations du kien:** `OWNS`, `MENTIONS`, `DEPENDS_ON`, `APPROVES`, `UPDATED_BY`, `USES`.
- **Entity Resolution:** manual alias cho phong ban/san pham, vector candidate search, lexical guard, audit log.
- **Super-node:** cac node nhu `Company`, `Policy`, `HR` se cap theo date/type va dung community partition.

---

## TU DANH GIA

| Tieu chi | Diem tu cham (1-5) | Ghi chu |
|---|---:|---|
| Muc do hieu GraphRAG | 4 | Hieu pipeline va trade-off Flat vs Graph. |
| Kiem soat AI Coding Agent | 4 | Co dieu chinh local, khong hard-code secret, co ghi fallback. |
| Chat luong graph | 3 | Graph dung provenance nhung nho do limit token. |
| Phan tich/debug he thong | 4 | Xu ly gated dataset, local path, Neo4j stale, Groq limit. |
