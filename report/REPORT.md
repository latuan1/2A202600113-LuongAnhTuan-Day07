# Báo Cáo Lab 7: Embedding & Vector Store

**Họ và tên:** Lương Anh Tuấn

**MSV:** 2A202600113

**Nhóm:** C401-E5

---

## Section 1. Warm-up (5 điểm)

### Exercise 1.1 — Cosine Similarity in Plain Language

**What does it mean for two text chunks to have high cosine similarity?**
> Hai đoạn văn bản có hướng embedding gần giống nhau, tức là nội dung hoặc ý nghĩa của chúng rất giống nhau.

**Give a concrete example of two sentences that would have HIGH similarity and two that would have LOW similarity.**
**A concrete example of two sentences that would have HIGH similarity:**
- Sentence A: Tôi thích học lập trình Python.
- Sentence B: Lập trình Python là sở thích của tôi.
- Tại sao tương đồng: Cùng nói về sở thích học lập trình Python, ý nghĩa gần như giống hệt.

**A concrete example of two sentences that would have LOW similarity:**
- Sentence A: Trời hôm nay rất đẹp.
- Sentence B: Tôi đang học lập trình Python.
- Tại sao khác: Một câu nói về thời tiết, một câu nói về lập trình, ý nghĩa hoàn toàn khác nhau.

**Why is cosine similarity preferred over Euclidean distance for text embeddings?**
> Cosine similarity đo lường sự tương đồng về hướng (ý nghĩa) thay vì độ lớn, phù hợp hơn cho so sánh văn bản vì độ dài vector embedding không quan trọng bằng ý nghĩa.

### Chunking Math (Ex 1.2)

**A document is 10,000 characters. You chunk it with `chunk_size=500`, `overlap=50`. How many chunks do you expect?**
> Phép tính: num_chunks = ceil((10,000 - 50) / (500 - 50)) = ceil(9950 / 450) ≈ 23.22 → 24 chunks.
> Đáp án: 24 chunks.

**If overlap is increased to 100, how does this change the chunk count? Why would you want more overlap?**
> Chunk count sẽ tăng lên vì mỗi chunk mới bắt đầu gần chunk trước hơn (num_chunks = ceil((10,000 - 100) / (500 - 100)) = ceil(9900 / 400) = 25). Overlap nhiều hơn giúp giữ ngữ cảnh liền mạch giữa các chunk, giảm nguy cơ mất thông tin quan trọng ở ranh giới chunk.

---

