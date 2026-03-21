---
layout: default
title: Home
---

# 구본승의 기술 블로그

Backend 개발, AI 개발 자동화, 하네스 엔지니어링에 대해 기록합니다.

---

## 최근 글

{% for post in site.posts limit:10 %}
### [{{ post.title }}]({{ post.url }})
<small>{{ post.date | date: "%Y-%m-%d" }} · {{ post.categories | join: ", " }}</small>

{{ post.excerpt }}

---
{% endfor %}

{% if site.posts.size == 0 %}
아직 작성된 글이 없습니다. 곧 첫 글을 올리겠습니다.
{% endif %}
