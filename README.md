# 🤖 FahMai RAG: Thai AI Customer Service (SuperAI SS6)

> 🏆 โปรเจคนี้เป็นผลงานจากการแข่งขัน Mini Hackathon ในโครงการ **SuperAI Engineer Season 6**

> ภารกิจคือการสร้างระบบ Retrieval-Augmented Generation (RAG) สำหรับ "ร้านฟ้าใหม่ (FahMai)" ซึ่งเป็นร้านขายอุปกรณ์อิเล็กทรอนิกส์จำลอง เพื่อทำหน้าที่เป็น AI Customer Service ตอบคำถามลูกค้า 100 ข้อ โดยใช้โมเดลจาก ThaiLLMs

## 📌 Concept
ระบบ RAG นี้ถูกออกแบบมาเพื่อดึงข้อมูลจาก Knowledge Base ของร้านฟ้าใหม่ (เอกสาร Markdown 118 ไฟล์ ทั้งข้อมูลสินค้า นโยบายร้าน และสาขา) เพื่อนำมาตอบคำถามแบบปรนัย (Multiple-choice) 

ความท้าทายหลักของโปรเจกต์นี้คือ:
- การรับมือกับคำถามที่ **ไม่มีข้อมูลในฐานข้อมูล** (ต้องตอบ Choice 9)
- การรับมือกับคำถามที่ **ไม่เกี่ยวข้องกับร้านฟ้าใหม่** (ต้องตอบ Choice 10)
- คำถาม Test ที่มีความซับซ้อนและไม่ได้ถามตรงตัวจากสินค้า เช่น การให้โจทย์เป็นสเปกสินค้าที่ต้องการ 

---

## 🏗️ Architecture & Pipeline Design
ระบบถูกพัฒนาขึ้นด้วยแนวคิด **Advanced RAG Pipeline** เพื่อเพิ่มความแม่นยำสูงสุดในการค้นหาข้อมูล:

### 🔹 Phase 1: Advanced Document Chunking
- หั่นเอกสารโดยรักษาบริบทของหัวข้อด้วย `MarkdownHeaderTextSplitter`
- แบ่งเนื้อหาย่อยด้วย `RecursiveCharacterTextSplitter` พร้อมใส่ Metadata (หมวดหมู่, ชื่อไฟล์, หัวข้อย่อย) กำกับไว้ในทุก Chunk เพื่อให้โมเดลไม่หลงบริบท

### 🔹 Phase 2: Hybrid Retrieval & Re-ranking
- **Dense Retrieval (FAISS):** ทำ Semantic Search ค้นหาด้วยความหมายผ่านโมเดล Embeddings `intfloat/multilingual-e5-large`
- **Sparse Retrieval (BM25):** ทำ Keyword Search ตัดคำภาษาไทยเพื่อหาคำที่ตรงกันเป๊ะๆ (Exact match) ป้องกันปัญหาการค้นหารหัสรุ่นสินค้าผิดพลาด
- **Reciprocal Rank Fusion (RRF):** รวมผลลัพธ์จากการค้นหาทั้ง 2 แบบเข้าด้วยกันเพื่อดึงจุดแข็งของทั้งคู่
- **Re-ranking:** ใช้ Cross-Encoder (`BAAI/bge-reranker-v2-m3`) จัดเรียงลำดับความสำคัญของเนื้อหาใหม่ให้แม่นยำที่สุด ก่อนส่งให้ LLM

### 🔹 Phase 3: Chain-of-Thought (CoT) Prompting & Generation
- ส่งข้อมูลที่คัดกรองแล้วไปยัง **ThaiLLMs** (ผ่าน API)
- ออกแบบ Prompt แบบให้ AI "คิดทีละสเตป" (Chain-of-Thought) เพื่อป้องกันการเดาคำตอบ (Hallucination) โดยบังคับให้พิจารณาก่อนว่าคำถามเกี่ยวข้องไหม มีข้อมูลพอไหม ก่อนที่จะเทียบกับตัวเลือก 1-8

---

## ✨ Key Features & Technical Highlights
- **Hybrid Search Engine:** เอาชนะข้อจำกัดของ Vector Search ทั่วไปที่มักจะพลาดเมื่อเจอคำค้นหาที่เป็นรหัสสินค้าเฉพาะเจาะจง
- **Automated Fallback to Edge Cases:** ตัว Prompt ถูกออกแบบมาเพื่อดักจับ Out-of-domain query ทำให้ตอบข้อ 9 และ 10 ได้อย่างมั่นใจ
- **Resilient API Calling:** มีระบบ Retry อัตโนมัติด้วยไลบรารี `tenacity` เพื่อป้องกันปัญหา API ล่มหรือติด Rate Limit ระหว่างการรันผลจำนวนมาก
- **Checkpoint & Error Analysis:** ระบบบันทึกผลลัพธ์แบบ JSONL ทุกครั้งที่ทำเสร็จแต่ละข้อ ช่วยให้สามารถรันต่อจากจุดที่ค้างได้ทันที และนำ Log มาวิเคราะห์เหตุผลที่ AI ตอบผิดได้

---

## 🛠️ Tech Stack

- **Language:** Python 3 (รันและพัฒนาบน Google Colab)
- **RAG Framework & Retrieval:** `LangChain`, `FAISS`, `rank_bm25`
- **NLP & Models:** `HuggingFaceEmbeddings`, `sentence-transformers` (CrossEncoder), `pythainlp` (Thai Tokenization)
- **LLM:** API `ThaiLLMs`
- **Utilities:** `pandas`, `tenacity` (Retry mechanism), `tqdm` (Progress tracking)

---

## 🚀 How to Run

เนื่องจากไม่มีโฟลเดอร์ Data ให้ หากต้องการนำไปรันเพื่อทดสอบ จำเป็นต้องเตรียมโครงสร้างโฟลเดอร์ให้ถูกต้องดังนี้:

### 1. โครงสร้างโฟลเดอร์ (Directory Structure)
```text
├── data/
│   ├── questions.csv                           # ไฟล์คำถามที่ต้องตอบ
│   └── (โฟลเดอร์ Knowledge Base ทั้งหมด)
├── output/                                     # โฟลเดอร์สำหรับเก็บผลลัพธ์และ log
└── hybrid-RAG-thai-customer-service.ipynb      # โค้ดหลักของ RAG Pipeline
```

### 2. ติดตั้ง Dependencies
```sh
pip install pandas tqdm pythainlp rank_bm25 langchain langchain-community tenacity sentence-transformers faiss-cpu openai langchain-text-splitters langchain-huggingface
```

### 3. ตั้งค่า API Key
ตั้งค่า Environment Variable สำหรับ API Key ของ ThaiLLM
```sh
export THAILLM_API_KEY="your-api-key-here"
```

### 4. รัน Pipeline
เปิดไฟล์ Jupyter Notebook เพื่อเริ่มกระบวนการ สร้าง Index และรัน Inference ระบบจะส่งออกไฟล์ `submission.csv` และไฟล์ `analysis_log.csv` สำหรับตรวจสอบความแม่นยำ