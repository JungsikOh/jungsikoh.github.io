---
layout: page
title: Blog
permalink: /blog/
---

<style>
  /* 부모 컨테이너에 Flex + 오른쪽 정렬 */
  .search-container {
    display: flex;
    justify-content: flex-end; /* 자식 요소(입력창)를 오른쪽에 배치 */
    margin-bottom: 1rem;
  }

  /* 입력창 자체 스타일 */
  .search-input {
    width: 100%;
    max-width: 200px;
    padding: 0.6rem 1rem;
    font-size: 1rem;
    border: 1px solid #ccc;
    border-radius: 1.5rem;
    outline: none;
    box-sizing: border-box;
    transition: border-color 0.2s ease-in-out;
  }
  .search-input:focus {
    border-color: #6593F5;
  }
.post-tags {
  margin-left: 1rem; /* 혹시 왼쪽에 간격을 주고 싶다면 */
}

.post-tag {
  background-color: #f0f0f0; /* 연한 회색 배경 */
  color: #333;               /* 글자색 */
  padding: 3px 6px;         /* 안쪽 여백 */
  margin-left: 8px;         /* 태그 간 간격 */
  border-radius: 9999px;    /* 완전 둥글게 */
  font-size: 0.85rem;       /* 조금 작게 */
  white-space: nowrap;      /* 태그가 두 줄로 끊기지 않도록 */
}

</style>

<section class="c-archives">
  <!-- 검색 입력 상자 -->
  <div class="search-container">
  <input
    type="text"
    id="searchInput"
    class="search-input"
    placeholder="Search by title or tags..."
  />
  </div>

  {% for post in site.posts %}
    {% capture this_year %}{{ post.date | date: "%Y" }}{% endcapture %}
    {% capture next_year %}{{ post.previous.date | date: "%Y" }}{% endcapture %}

    <!-- 연도 헤더 표시 로직은 그대로 유지 -->
    {% if forloop.first %}
      <h2 class="c-archives__year" id="{{ this_year }}-ref">{{ this_year }}</h2>
      <ul class="c-archives__list" id="postList">
    {% endif %}

    <li
      class="c-archives__item"
      data-title="{{ post.title | downcase }}"
      data-tags="{% for tag in post.tags %}{{ tag | downcase }} {% endfor %}"
    >
      <h3>
        <a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        <br>
        <small>{{ post.description }}</small>
        <!-- 오른쪽: 태그 목록 -->
       <span class="post-tags">
       {% for tag in post.tags %}
        <span class="post-tag">{{ tag }}</span>
       {% endfor %}
       </span>
      </h3>
      <p>{{ post.date | date: "%b %-d, %Y" }}</p>
    </li>

    {% if forloop.last %}
      </ul>
    {% else %}
      {% if this_year != next_year %}
      </ul>
      <h2 class="c-archives__year" id="{{ next_year }}-ref">{{ next_year }}</h2>
      <ul class="c-archives__list" id="postList">
      {% endif %}
    {% endif %}
  {% endfor %}
</section>

<!-- (B) 자바스크립트 필터 로직 -->
<script>
document.addEventListener("DOMContentLoaded", function() {
  const searchInput = document.getElementById("searchInput");
  const postItems = document.querySelectorAll(".c-archives__item");

  searchInput.addEventListener("input", function() {
    const query = this.value.trim().toLowerCase();
    postItems.forEach((item) => {
      const title = item.getAttribute("data-title");
      const tags = item.getAttribute("data-tags");
      if (title.includes(query) || tags.includes(query)) {
        item.style.display = "";
      } else {
        item.style.display = "none";
      }
    });
  });
});
</script>