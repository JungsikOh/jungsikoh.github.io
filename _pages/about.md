---
layout: page
title: Blog
permalink: /blog/
---

<style>
  /* 기존 스타일 유지 */
  .search-container {
    display: flex;
    justify-content: flex-end;
    margin-bottom: 1rem;
  }
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
  .post-tags { margin-left: 1rem; }
  .post-tag {
    background-color: #f0f0f0;
    color: #333;
    padding: 3px 6px;
    margin-left: 8px;
    border-radius: 9999px;
    font-size: 0.85rem;
    white-space: nowrap;
  }

  /* [새로 추가] 아코디언 스타일 */
  details.year-archive {
    margin-bottom: 1.5rem;
    border-bottom: 1px solid #eee;
  }
  details.year-archive > summary {
      display: flex;              /* Flexbox 사용 */
      align-items: center;        /* 세로축 중앙 정렬 */
      justify-content: space-between; /* 양끝 정렬 (왼쪽:연도, 오른쪽:아이콘) */
      cursor: pointer;
      outline: none;
      list-style: none;
      padding: 10px 0;
    }
  /* 브라우저 기본 화살표 숨기기 */
  details.year-archive > summary::-webkit-details-marker {
    display: none;
  }
  /* 연도 제목 스타일 */
  .c-archives__year {
    margin: 0;
    line-height: 1.2;
  }
  /* 우측 + / - 아이콘 */
  details.year-archive > summary::after {
      content: '+'; 
      font-size: 2.5rem;    /* [변경] 아이콘 크기 확대 (1.5rem -> 2.5rem) */
      font-weight: 300;     /* 아이콘이 커진 만큼 두께는 살짝 얇게 (선택사항) */
      color: #ccc;
      margin-left: 10px;    /* 글자와의 최소 간격 */
      
      /* [중요] 위치 미세 조정 */
      line-height: 1;       /* 아이콘 자체의 줄높이를 줄여 박스 크기 최소화 */
      margin-top: 4px;      /* [변경] 아이콘을 아래로 살짝 내림 (수치 조절 가능) */
    }
  details.year-archive[open] > summary::after {
    content: '-';
    font-size: 2.5rem;
  }
</style>

<section class="c-archives">
  <div class="search-container">
    <input
      type="text"
      id="searchInput"
      class="search-input"
      placeholder="Search by title or tags..."
    />
  </div>

  {% assign current_year = "now" | date: "%Y" %}
  {% assign postsByYear = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}

  {% for year_group in postsByYear %}
    {% assign year = year_group.name %}

    <details class="year-archive" {% if year == current_year %}open{% endif %}>
      
      <summary>
        <h2 class="c-archives__year" id="{{ year }}-ref">{{ year }}</h2>
      </summary>

      <ul class="c-archives__list" id="list-{{ year }}">
        {% for post in year_group.items %}
        <li
          class="c-archives__item"
          data-title="{{ post.title | downcase }}"
          data-tags="{% for tag in post.tags %}{{ tag | downcase }} {% endfor %}"
        >
          <h3>
            <a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
            <br>
            <small>{{ post.description }}</small>
            
            <span class="post-tags">
              {% for tag in post.tags %}
                <span class="post-tag">{{ tag }}</span>
              {% endfor %}
            </span>
          </h3>
          <p>{{ post.date | date: "%b %-d, %Y" }}</p>
        </li>
        {% endfor %}
      </ul>

    </details>
  {% endfor %}

</section>

<script>
document.addEventListener("DOMContentLoaded", function() {
  const searchInput = document.getElementById("searchInput");
  // 모든 details와 리스트 아이템 가져오기
  const allDetails = document.querySelectorAll("details.year-archive");
  const postItems = document.querySelectorAll(".c-archives__item");

  searchInput.addEventListener("input", function() {
    const query = this.value.trim().toLowerCase();

    // 1. 검색어가 없으면: 초기 상태로 복구 (올해만 펴고 나머지는 접음)
    if (query === "") {
      const currentYear = new Date().getFullYear().toString();
      allDetails.forEach(detail => {
        const yearText = detail.querySelector(".c-archives__year").innerText.trim();
        if (yearText === currentYear) {
          detail.open = true;
        } else {
          detail.open = false;
        }
        // 모든 아이템 다시 표시
        detail.querySelectorAll(".c-archives__item").forEach(item => item.style.display = "");
      });
      return;
    }

    // 2. 검색어가 있으면 필터링 수행
    postItems.forEach((item) => {
      const title = item.getAttribute("data-title");
      const tags = item.getAttribute("data-tags");
      const parentDetails = item.closest("details"); // 부모 details 찾기

      if (title.includes(query) || tags.includes(query)) {
        item.style.display = ""; // 검색어 포함되면 보이기
        if (parentDetails) {
          parentDetails.open = true; // [중요] 검색 결과가 있는 연도는 자동으로 펼침
        }
      } else {
        item.style.display = "none"; // 안 맞으면 숨기기
      }
    });
    
    // (선택사항) 검색 결과가 하나도 없는 연도는 접고 싶다면 추가 로직이 필요하지만, 
    // 위 로직만으로도 검색된 글이 있는 연도는 무조건 펼쳐지므로 사용성이 충분합니다.
  });
});
</script>