# 🟢 Candidate Data

* <mark style="color:purple;background-color:purple;">**Notice the dense\_vector data type used here**</mark>
* <mark style="color:purple;background-color:purple;">**We are having candidate metadata + resume text + resume embedding**</mark>

```json
PUT /candidates
{
  "mappings": {
    "properties": {
      "candidate_id": { "type": "keyword" },
      "name": { "type": "text" },
      "email": { "type": "keyword" },
      "phone": { "type": "keyword" },
      "experience_years": { "type": "integer" },
      "experience_level": { "type": "keyword" },
      "current_role": { "type": "text" },
      "role_category": { "type": "keyword" },
      "current_company": { "type": "keyword" },
      "company_sector": { "type": "keyword" },
      "location": { "type": "keyword" },
      "city": { "type": "keyword" },
      "state": { "type": "keyword" },
      "skills": { "type": "keyword" },
      "primary_skills": { "type": "keyword" },
      "secondary_skills": { "type": "keyword" },
      "education": { "type": "text" },
      "has_overlap": { "type": "boolean" },
      "secondary_role": { "type": "keyword" },
      "overlap_domains": { "type": "keyword" },
      "resume_text": { "type": "text" },
      "summary": { "type": "text" },
      "ctc": { "type": "float" },
      "notice_period_days": { "type": "integer" },
      "resume_upload_date": { "type": "keyword" },
      "team_size_managed": { "type": "integer" },
      "resume_embedding": {
        "type": "dense_vector",
        "dims": 384,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}
```
