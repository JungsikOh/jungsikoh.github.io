---
layout: page
title: Blog
permalink: /blog/
---
<section class="c-archives">
  <!-- 검색 입력 상자 -->
  <input
    type="text"
    id="searchInput"
    placeholder="Search by title or tags..."
    style="margin-bottom: 1rem; width: 100%; max-width: 300px; padding: 0.5rem;"
  />

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

---
layout: page
title: "Blog"
permalink: /blog/
---

<section class="c-archives">
  <!-- (A) 포스트 목록 생성 부분 (data-title, data-tags 등) -->
  <input type="text" id="searchInput" placeholder="Search..."/>
  <ul id="postList">
    {% for post in site.posts %}
    <li class="c-archives__item"
        data-title="{{ post.title | downcase }}"
        data-tags="{% for tag in post.tags %}{{ tag | downcase }} {% endfor %}">
      <h3>{{ post.title }}</h3>
    </li>
    {% endfor %}
  </ul>
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