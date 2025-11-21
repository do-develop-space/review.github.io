---
layout: home
title: 개발 블로그
---

# 개발 블로그에 오신 것을 환영합니다! 👋

이 블로그는 DDD(Domain-Driven Design)와 헥사고날 아키텍처를 적용한 Spring Boot 이커머스 스토어 프로젝트의 개발 과정을 기록합니다.

## 최근 포스트

<div class="posts">
  {% for post in site.posts limit:5 %}
    <article class="post">
      <h2><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h2>
      <p class="post-date">{{ post.date | date: "%Y년 %m월 %d일" }}</p>
      <p>{{ post.excerpt }}</p>
    </article>
  {% endfor %}
</div>

## 프로젝트

- **[Yellow Store](https://github.com/do-develop-space/yellow-store)**: DDD와 헥사고날 아키텍처로 구현한 Spring Boot 이커머스 스토어

## 기술 스택

- Spring Boot 4.0.0
- Java 17
- PostgreSQL
- JPA
- DDD (Domain-Driven Design)
- 헥사고날 아키텍처 (Ports & Adapters)

