# 개발 블로그

GitHub Pages를 통해 호스팅되는 Jekyll 블로그입니다.

## 로컬에서 블로그 실행하기

### 1. Ruby와 Jekyll 설치

```bash
# macOS (Homebrew 사용)
brew install ruby

# 또는 rbenv 사용
rbenv install 3.1.0
rbenv global 3.1.0
```

### 2. Jekyll과 의존성 설치

```bash
gem install bundler jekyll
bundle install
```

### 3. 로컬 서버 실행

```bash
bundle exec jekyll serve
```

브라우저에서 `http://localhost:4000`으로 접속하면 블로그를 확인할 수 있습니다.

## 새 포스트 작성하기

`_posts` 폴더에 다음 형식으로 파일을 생성하세요:

```
YYYY-MM-DD-포스트-제목.md
```

예시:
```markdown
---
layout: post
title: "DDD 아키텍처 적용기"
date: 2024-12-21
categories: [아키텍처]
tags: [DDD, 헥사고날, Spring Boot]
---

# DDD 아키텍처 적용기

본문 내용...
```

## GitHub Pages 설정

1. GitHub 저장소의 Settings → Pages로 이동
2. Source에서 "Deploy from a branch" 선택
3. Branch를 `main`으로, Folder를 `/ (root)`로 설정
4. Save 클릭

설정이 완료되면 `https://do-develop-space.github.io/review.github.io`에서 블로그를 확인할 수 있습니다.

## GitHub Actions 자동 배포

`.github/workflows/pages.yml` 파일이 포함되어 있어, `main` 브랜치에 푸시하면 자동으로 블로그가 배포됩니다.

