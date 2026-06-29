# 좋아요(liked) 일괄 가져오기 — 전 센터 확장 밑작업

작성일: 2026-06-29 · 브랜치: `feat/liked-scan-groundwork`

## 목적
melting/zeta/tingle에만 있던 "♥ 좋아요 목록 일괄 가져오기"를 나머지 센터
(loveydovey·babechat·tikita·rofan·chub·whif)로 확장. 실제 동작엔 각 사이트 **로그인 세션/쿠키 +
좋아요 API 엔드포인트**가 필요하므로, 그 전까지 공통 뼈대만 깔아두고 비활성(스텁)으로 보관한다.
세션 확보 시 **어댑터 본문만 채우고 페이지에 배선**하면 켜진다.

## 동작 구조 (작동 중 3센터 기준)
유저 인증 → `getGlobalConfigValue(키)`로 저장된 자격증명 로딩 → 외부 사이트 좋아요 API 호출 →
`LikedItem{ id,name,coverImageUrl,tags,isAdult?,sourceUrl }[]`로 매핑 → `{ liked, total }` 반환.
페이지는 공통 바텀시트로 목록을 보여주고 선택 → `/api/characters/import`로 일괄 가져오기.

## 이번에 만든 공통 자산 (활성)
- `lib/likedScan.ts` — `LikedItem` 타입 + `runLikedScan(req, scan)` 공통 래퍼 + `LikedScanError`.
  파일 하단에 **신규 센터 어댑터 예시(주석)** 보관(babechat=Bearer, whif=Cookie 등).
- `components/ui/LikedImportSheet.tsx` — 공통 좋아요 바텀시트(테마는 `prefix` prop). melting/zeta/tingle을
  이걸로 리팩터해 ~250줄 중복 제거(동작 동일).
- `app/api/{loveydovey,babechat,tikita,rofan,chub,whif}/liked-scan/route.ts` — **501 스텁**(준비 중).

## 센터별 현황
| 센터 | 자격증명(GlobalConfig 키) | 좋아요 API | 비고 |
|---|---|---|---|
| melting · zeta · tingle | 보유 | 구현됨(작동) | 공통 시트로 리팩터 완료 |
| **babechat** | `babechat_access_token`(+refresh) 보유 | 미상 | Bearer 추정 — 엔드포인트만 찾으면 빠름 |
| **whif** | `whif_session_cookie` 보유 | 미상 | Cookie 방식(captureWhif와 동일 쿠키) |
| loveydovey · tikita · rofan · chub | 없음 | 미상 | 세션 쿠키/토큰을 admin/import-cookies에 추가 저장부터 |

## 활성화 체크리스트 (센터 1개 기준)
1. **자격증명**: 없으면 `admin/import-cookies`에 입력칸 추가 + `CookieData` 타입/`/api/admin/import-cookies`에 키 추가.
   (babechat·whif는 이미 있음)
2. **스캔 어댑터**: `app/api/<center>/liked-scan/route.ts` 스텁을 아래로 교체(엔드포인트·응답구조·sourceUrl 확정):
   ```ts
   import { NextRequest } from 'next/server'
   import { runLikedScan, LikedScanError, type LikedItem } from '@/lib/likedScan'
   import { getGlobalConfigValue } from '@/lib/import/capture'
   export async function GET(req: NextRequest) {
     return runLikedScan(req, async () => {
       const cred = await getGlobalConfigValue('<center>_session_cookie') // 또는 _access_token
       if (!cred) throw new LikedScanError('<center> 자격증명이 없습니다.', 400)
       const res = await fetch('https://<api>/<좋아요 엔드포인트>', {
         headers: { Cookie: cred /* 또는 Authorization: `Bearer ${cred}` */, Accept: 'application/json' },
       })
       if (res.status === 401 || res.status === 403) throw new LikedScanError('<center> 세션 만료.', 400)
       if (!res.ok) throw new LikedScanError(`<center> API 오류 (HTTP ${res.status})`)
       const data = await res.json()
       const items: any[] = data?.<배열경로> ?? []
       const liked: LikedItem[] = items.map(x => ({
         id: String(x.id), name: x.name ?? '',
         coverImageUrl: x.<이미지> ?? null,
         tags: (x.tags ?? []).map((t: any) => t?.name ?? t),
         isAdult: !!x.isAdult,
         sourceUrl: `https://<center>/<경로>/${x.id}`,
       }))
       return { liked }
     })
   }
   ```
3. **페이지 배선**: 해당 센터 `app/(<center>)/<center>/page.tsx`에 아래를 추가(melting 페이지가 동일 패턴 — 그대로 참고).
   - import: `import LikedImportSheet from '@/components/ui/LikedImportSheet'`
   - 상태:
     ```tsx
     const [likedPanel, setLikedPanel] = useState(false)
     const [likedList, setLikedList] = useState<{ id:string;name:string;coverImageUrl:string|null;tags:string[];isAdult?:boolean;sourceUrl:string }[]>([])
     const [likedSelected, setLikedSelected] = useState<Set<string>>(new Set())
     const [scanning, setScanning] = useState(false)
     const [scanMsg, setScanMsg] = useState('')
     ```
     (`importing`/`msg`는 기존 import 상태 재사용)
   - 핸들러: melting의 `handleLikedScan`(엔드포인트만 `/api/<center>/liked-scan`로)·`handleLikedImport` 복사.
   - 메뉴: 가져오기 메뉴에 `<button className="<center>-menu-item" onClick={handleLikedScan}>♥ 좋아요 목록</button>`
   - 시트: 컴포넌트 배치
     ```tsx
     <LikedImportSheet open={likedPanel} onClose={() => setLikedPanel(false)}
       title="♥ <센터> 좋아요 목록" prefix="<테마접두사>"
       items={likedList} scanning={scanning} scanMsg={scanMsg}
       onRescan={() => { setLikedList([]); setLikedSelected(new Set()); setScanMsg(''); handleLikedScan() }}
       alreadyImported={x => items.some(c => c.sourceUrl === x.sourceUrl)}
       selected={likedSelected} onChangeSelected={setLikedSelected}
       importing={importing} importProgress={msg} onImport={handleLikedImport} />
     ```
   - 테마 접두사: loveydovey=`l`, babechat=`b`, tikita=`t`, rofan=`r`, chub=`c`, whif=`w`, melting=`m`, zeta=`z`, tingle=`tg`.
4. **가이드 동기화**: 사용자 체감 기능이므로 `app/(main)/guide/page.tsx`에 반영.

## 비목표(YAGNI)
- 세션 없이 동작하는 우회/헤드리스 — 기존 정책상 API 직접호출 유지.
- 센터별 페이지 배선을 미리 주석으로 심는 것(작동 중 리스트 페이지 오염 방지) — 위 스니펫으로 활성화 시 붙임.
