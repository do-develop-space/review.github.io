---
layout: home
title: 개발 블로그
---

# 개발 블로그에 오신 것을 환영합니다! 👋

이 블로그는 DDD(Domain-Driven Design)와 헥사고날 아키텍처를 적용한 Spring Boot 이커머스 스토어 프로젝트의 개발 과정을 기록합니다.

<div class="posts">
  {% assign sorted_posts = site.posts | sort: 'date' | reverse %}
  {% for post in sorted_posts limit:5 %}
    <article class="post">
      <h2><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h2>
      <p class="post-date">{{ post.date | date: "%Y년 %m월 %d일" }}</p>
      <p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">더 보기 →</a>
    </article>
  {% endfor %}
</div>

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

