# 발표 예상 질문 (FAQ)

발표·심사에서 나올 만한 기술 질문과, 실제 코드·공식 문서로 확인한 답을 모아 둔다.
시스템 구성 자체에 대한 설명은 [기술 및 서비스 구성](tech-stack.md)을 참고.

---

## 왜 Cloudflare Pages인가? 정적 파일뿐이면 GitHub Pages로도 되지 않나?

**짧은 답** — 앱 자체는 GitHub Pages에서도 그대로 돌아간다. 고쳐야 할 코드는 없다.
다만 ① 레포가 private이라 무료 플랜으로는 쓸 수 없고, ② 앱이 들어 있는 `www/` 폴더를
발행 폴더로 지정할 수 없어서 Cloudflare Pages를 골랐다. 앱의 문제가 아니라 호스팅 쪽 제약이었다.

### 앱 쪽은 호환 문제가 없다

하위 경로 배포(`user.github.io/HeartGuardiansApp/`)에서 제일 자주 깨지는 게 경로인데, 이 앱은 해당 없다.

| 항목                              | 확인 결과                                                                 |
| --------------------------------- | ------------------------------------------------------------------------- |
| 루트 절대경로(`src="/assets/..."`) | **0건** — 모든 경로가 상대경로 (`ROOT`가 페이지 깊이별로 `""` / `"../"` / `"../../"`) |
| 페이지 이동                       | `location.href = ROOT + "auth/index.html"` — 전부 상대경로                 |
| API 호출                          | Workers 주소를 **절대 URL**로 호출 + CORS 와일드카드 → 호스팅 위치와 무관   |
| SPA 폴백(`_redirects`) 필요 여부  | 불필요 — MPA라 모든 URL에 실제 `index.html` 파일이 존재                     |
| 사이트 용량                       | 78 MB (436개 파일, 최대 파일 10.9 MB) → Pages 상한 **1 GB** 안쪽            |

### 걸리는 것 세 가지

**1. private 레포 — 가장 큰 벽**

> "If the account that owns the repository uses GitHub Free or GitHub Free for organizations,
> the repository must be public."
> — [GitHub Docs · Creating a GitHub Pages site](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site)

`HeartGuardiansApp`은 private이다. 무료 플랜이면 **레포를 공개로 돌리거나 유료 플랜(Pro)으로
올려야** 한다. 게다가 게시된 사이트는 레포가 private이어도 **인터넷에 그대로 공개된다**
(접근 제어는 Enterprise Cloud 전용). Cloudflare Pages는 private 레포를 무료로 연결할 수 있고,
필요하면 Cloudflare Access로 접근 제한도 걸 수 있다.

**2. 발행 폴더를 `www/`로 지정할 수 없다**

GitHub Pages는 브랜치 배포 시 소스 폴더로 **저장소 루트(`/`) 또는 `/docs`** 둘만 허용한다.
이 앱은 `www/`에 있고, 그 이름은 Capacitor의 `webDir: "www"`와 묶여 있다. 그래서 셋 중 하나를 해야 한다.

- `www/` → `docs/`로 이름 변경 (+ `capacitor.config.json` 수정) — 문서 폴더와 이름이 겹쳐 혼란
- `gh-pages` 브랜치에 `www/` 내용만 따로 푸시
- GitHub Actions 워크플로로 `www/`를 업로드

Cloudflare Pages는 대시보드 output 디렉터리에 **`www`라고 적으면 끝**이고 빌드 명령도 없다.
(이 프로젝트는 예전에 GitHub Actions 배포를 시도했다가 되돌린 적이 있다 — 배포 자격증명을
Cloudflare 한쪽에 두는 편이 단순해서였다.)

**3. 대역폭**

GitHub Pages는 월 **100 GB 소프트 제한**이다. 캐시 없는 풀 로드가 78 MB니까 단순 계산으로
월 약 **1,300회** — 30명 학급 기준 약 43개 학급분. 동영상·3D 모델이 무거운 앱이라 여유가 많지 않다.
Cloudflare Pages는 대역폭 무제한이고 애초에 CDN 제품이다.
([GitHub Pages 사용 한도](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits))

덧붙여 GitHub Pages 약관에는 상업적 서비스 호스팅 금지 조항이 있다. 교육용이라 문제될 일은
아니지만 사업화한다면 걸린다.

### APK는 호스팅과 무관하다

`www/` 전체가 APK 안에 들어가므로, 태블릿에서 돌아가는 최종 앱은 **어디에 호스팅하든 상관없다.**
웹 호스팅은 "브라우저로도 해 볼 수 있게" 하는 부가 경로일 뿐이다.

### 비교

|                          | GitHub Pages                 | Cloudflare Pages (채택) |
| ------------------------ | ---------------------------- | ----------------------- |
| 정적 파일 서빙           | 가능                         | 가능                    |
| private 레포             | 무료 플랜 불가 (공개 or 유료) | 가능                    |
| 출력 폴더 `www` 지정     | 불가 (`/` 또는 `/docs`만)     | 대시보드에서 지정       |
| 대역폭                   | 100 GB/월 소프트 제한         | 무제한                  |
| 이 앱의 경로·API 호환성  | 문제없음                      | 문제없음                |
