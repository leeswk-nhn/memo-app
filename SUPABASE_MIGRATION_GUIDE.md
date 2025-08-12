# 메모 앱 Supabase 마이그레이션 가이드

이 가이드는 메모 앱을 로컬 스토리지에서 Supabase 데이터베이스로 마이그레이션하는 방법을 설명합니다.

## 📋 마이그레이션 개요

- **기존**: localStorage 기반 로컬 데이터 저장
- **신규**: Supabase PostgreSQL 데이터베이스 기반 클라우드 저장
- **인터페이스**: 기존 타입 정의 그대로 유지
- **기능**: 모든 CRUD 기능 동일하게 동작

## 🔧 설정 단계

### 1. Supabase 프로젝트 생성

1. [Supabase 대시보드](https://supabase.com/dashboard)에 접속
2. 새 프로젝트 생성
3. 데이터베이스 비밀번호 설정

### 2. 환경 변수 설정

프로젝트 루트에 `.env.local` 파일을 생성하고 다음 값들을 입력하세요:

```env
# Supabase 프로젝트 URL
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url

# Supabase Anon Key (공개 키)
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
```

Supabase 대시보드의 Settings > API에서 이 값들을 확인할 수 있습니다.

### 3. 데이터베이스 스키마 생성

Supabase 대시보드의 SQL Editor에서 `supabase_migration.sql` 파일의 내용을 실행하세요.

## 📊 데이터베이스 스키마

### memos 테이블

```sql
CREATE TABLE public.memos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  category TEXT NOT NULL CHECK (category IN ('personal', 'work', 'study', 'idea', 'other')),
  tags TEXT[] DEFAULT ARRAY[]::TEXT[],
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 인덱스

- `idx_memos_created_at`: 생성일 기준 정렬을 위한 인덱스
- `idx_memos_category`: 카테고리별 필터링을 위한 인덱스

### 트리거

- `update_memos_updated_at`: 메모 수정 시 `updated_at` 자동 업데이트

## 🔒 보안 설정

### Row Level Security (RLS)

현재는 모든 작업을 허용하는 정책이 설정되어 있습니다. 프로덕션 환경에서는 사용자 인증에 따른 제한을 추가하세요.

```sql
-- 현재 정책 (개발용)
CREATE POLICY "Allow all operations on memos" ON public.memos
FOR ALL USING (true) WITH CHECK (true);

-- 추후 사용자별 제한 예시
-- CREATE POLICY "Users can only access their own memos" ON public.memos
-- FOR ALL USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);
```

## 🔄 변경된 구조

### 1. 새로운 파일들

- `src/lib/supabase.ts`: Supabase 클라이언트 설정
- `src/types/database.ts`: 데이터베이스 타입 정의
- `src/utils/supabaseUtils.ts`: Supabase 연동 유틸리티 함수

### 2. 업데이트된 파일들

- `src/hooks/useMemos.ts`: 비동기 데이터베이스 작업으로 변경
- `src/app/page.tsx`: 비동기 함수 호출 처리
- `src/components/MemoForm.tsx`: 비동기 폼 제출 처리
- `src/components/MemoItem.tsx`: 비동기 삭제 처리
- `src/components/MemoDetailModal.tsx`: 비동기 삭제 처리
- `src/utils/seedData.ts`: 데이터베이스용 샘플 데이터 시딩 추가

### 3. 주요 변경 사항

#### localStorage → Supabase 유틸리티 교체
```typescript
// 기존
import { localStorageUtils } from '@/utils/localStorage'

// 신규
import { supabaseUtils } from '@/utils/supabaseUtils'
```

#### 동기 → 비동기 함수 변경
```typescript
// 기존
const createMemo = (formData: MemoFormData): Memo => { ... }

// 신규
const createMemo = async (formData: MemoFormData): Promise<Memo | null> => { ... }
```

## 📦 의존성

새로 추가된 패키지:
- `@supabase/supabase-js`: Supabase JavaScript 클라이언트

```bash
npm install @supabase/supabase-js
```

## 🚀 실행 방법

1. 환경 변수 설정 완료
2. 데이터베이스 마이그레이션 실행
3. 개발 서버 시작:
   ```bash
   npm run dev
   ```

## 🔄 데이터 마이그레이션

기존 localStorage 데이터를 Supabase로 마이그레이션하려면:

1. 브라우저 개발자 도구에서 localStorage 데이터 확인
2. 필요시 기존 데이터를 JSON으로 추출
3. Supabase 대시보드에서 수동으로 데이터 입력 또는 SQL INSERT 문 사용

## 💡 주의사항

- 환경 변수가 올바르게 설정되지 않으면 앱이 작동하지 않습니다
- 첫 실행 시 샘플 데이터가 자동으로 생성됩니다
- 네트워크 연결이 필요합니다 (오프라인에서는 작동하지 않음)
- 에러 발생 시 브라우저 콘솔에서 로그를 확인하세요

## 📈 향후 개선사항

- 사용자 인증 및 권한 관리
- 실시간 동기화 (Supabase Realtime)
- 오프라인 지원 (PWA)
- 데이터 백업 및 복원 기능
- 파일 첨부 기능 (Supabase Storage)

## 🛠️ 트러블슈팅

### 환경 변수 오류
```
Missing Supabase environment variables
```
→ `.env.local` 파일 확인 및 올바른 값 설정

### 데이터베이스 연결 오류
```
Failed to load memos from database
```
→ Supabase 프로젝트 상태 확인 및 URL/키 재확인

### RLS 정책 오류
```
Row Level Security policy violation
```
→ 데이터베이스 정책 설정 확인