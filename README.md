# 🛠️ MRT Claude Skills

마이리얼트립 팀이 함께 만들고 사용하는 Claude 스킬 마켓플레이스입니다.

## 빠른 시작

### 1단계: CLI 설치 (처음 한 번만)

```bash
curl -fsSL https://raw.githubusercontent.com/doyounglee-myrealtrip/mrt-claude-skills/main/bin/mrt-skill -o /usr/local/bin/mrt-skill
chmod +x /usr/local/bin/mrt-skill
```

### 2단계: 스킬 사용하기

```bash
# 사용 가능한 스킬 목록 보기
mrt-skill list

# 카테고리별로 보기
mrt-skill list --category engineering

# 스킬 설치하기
mrt-skill install code-review

# 설치된 스킬 업데이트
mrt-skill update code-review

# 스킬 정보 보기
mrt-skill info code-review
```

## 스킬 카테고리

| 카테고리 | 설명 |
|----------|------|
| `engineering` | 개발팀용 스킬 (코드 리뷰, TDD 등) |
| `data-analytics` | 데이터 분석팀용 스킬 |
| `product` | 프로덕트팀용 스킬 |
| `contents-creating` | 콘텐츠 제작팀용 스킬 (블로그, SNS 등) |
| `common` | 전체 팀 공용 스킬 |

## 스킬 목록

전체 스킬 목록은 [registry.json](./registry.json)에서 확인하거나 `mrt-skill list`로 확인하세요.

## 새 스킬 추가하기

팀원이라면 누구나 스킬을 추가할 수 있어요! [CONTRIBUTING.md](./CONTRIBUTING.md)를 참고해주세요.

## 문의 / 피드백

각 스킬 폴더의 `skill-meta.json`에 있는 작성자에게 슬랙으로 문의하거나,
GitHub Issues에 남겨주세요.
