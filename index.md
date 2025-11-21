---
layout: home
title: 개발 블로그
---

# 개발 블로그에 오신 것을 환영합니다! 👋

이 블로그는 DDD(Domain-Driven Design)와 헥사고날 아키텍처를 적용한 Spring Boot 이커머스 스토어 프로젝트의 개발 과정을 기록합니다.

<button class="posts-toggle" onclick="togglePosts()">
  <span class="toggle-text">포스트 보기</span>
  <span class="toggle-icon">▼</span>
</button>

<div class="posts" id="posts-container" style="display: none;">
  {% for post in site.posts limit:5 %}
    <article class="post">
      <h2><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h2>
      <p class="post-date">{{ post.date | date: "%Y년 %m월 %d일" }}</p>
      <p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">더 보기 →</a>
    </article>
  {% endfor %}
</div>

<script>
function togglePosts() {
  const container = document.getElementById('posts-container');
  const button = document.querySelector('.posts-toggle');
  const toggleText = button.querySelector('.toggle-text');
  const toggleIcon = button.querySelector('.toggle-icon');
  
  if (container.style.display === 'none') {
    container.style.display = 'block';
    toggleText.textContent = '포스트 숨기기';
    toggleIcon.textContent = '▲';
    button.classList.add('active');
  } else {
    container.style.display = 'none';
    toggleText.textContent = '포스트 보기';
    toggleIcon.textContent = '▼';
    button.classList.remove('active');
  }
}
</script>

## 프로젝트

<div class="project-card">
  <h3><a href="https://github.com/do-develop-space/yellow-store">Yellow Store</a></h3>
  <p>DDD와 헥사고날 아키텍처로 구현한 Spring Boot 이커머스 스토어</p>
</div>

## 기술 스택

<div class="tech-stack">
  <span class="tech-badge">Spring Boot 4.0.0</span>
  <span class="tech-badge">Java 17</span>
  <span class="tech-badge">PostgreSQL</span>
  <span class="tech-badge">JPA</span>
  <span class="tech-badge">DDD</span>
  <span class="tech-badge">헥사고날 아키텍처</span>
</div>

