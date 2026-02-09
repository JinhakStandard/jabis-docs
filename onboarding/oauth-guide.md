# OAuth 인증 구현 가이드

이 문서는 JabisTemplate 프로젝트의 OAuth 인증 구현 내용을 정리한 가이드입니다.
다른 프로젝트에서 동일한 인증 방식을 구현할 때 참고하세요.

---

## 1. 인증 방식 개요

### 1.1 인증 프로세스
```
[사용자] → [로그인 버튼 클릭] → [통합인증 서버로 리다이렉트]
                                        ↓
[역할 선택 페이지] ← [콜백 페이지] ← [인증 완료 후 콜백 URL로 리다이렉트]
```

### 1.2 인증 확인 방식
- **서버 API 호출 방식**: `/oauth/userinfo` API를 호출하여 로그인 여부 확인
- 쿠키/토큰을 클라이언트에서 직접 관리하지 않음
- 서버에서 세션/쿠키를 관리하고, 클라이언트는 API 응답만으로 판단
- `withCredentials: true` 옵션으로 쿠키 자동 전송

### 1.3 페이지 구조
| 경로 | 페이지 | 설명 |
|------|--------|------|
| `/login` | LoginPage | 로그인 버튼 표시 |
| `/auth/callback` | CallbackPage | OAuth 콜백 처리 |
| `/` | SelectRolePage | 역할 선택 (인증 필요) |
| `/{role}/*` | 각 역할별 페이지 | 대시보드 등 (인증 필요) |

---

## 2. 파일 구조

```
src/
├── config/
│   └── authConfig.js       # OAuth 설정 (환경변수, 엔드포인트)
├── services/
│   └── authService.js      # 인증 서비스 (API 호출, 로그인/로그아웃)
├── stores/
│   └── authStore.js        # Zustand 인증 상태 관리
├── pages/
│   ├── LoginPage.jsx       # 로그인 페이지
│   ├── CallbackPage.jsx    # OAuth 콜백 페이지
│   └── SelectRolePage.jsx  # 역할 선택 페이지
└── .env                    # 환경변수
```

---

## 3. 환경변수 설정 (.env)

```env
# OAuth 모드 활성화 (true로 설정 시 통합 인증 사용)
VITE_OAUTH_ENABLED=true

# OAuth 클라이언트 ID
VITE_OAUTH_CLIENT_ID=your_client_id

# OAuth 콜백 URL (인증 완료 후 리다이렉트될 URL)
VITE_OAUTH_REDIRECT_URI=http://localhost:5174/auth/callback

# OAuth 스코프
VITE_OAUTH_SCOPES=openid profile email

# OAuth 인증 서버 URL
VITE_OAUTH_AUTH_SERVER=http://localhost:3000/api
```

---

## 4. 핵심 파일 구현

### 4.1 authConfig.js - OAuth 설정

```javascript
/**
 * OAuth 인증 설정
 */

// 인증 서버 URL (브라우저 리다이렉트용)
const AUTH_SERVER_URL = import.meta.env.VITE_OAUTH_AUTH_SERVER || 'https://auth.example.com'

// API 호출용 Base URL (개발 환경에서는 프록시 사용)
const API_BASE_URL = import.meta.env.DEV ? '/auth-api' : AUTH_SERVER_URL

export const AUTH_CONFIG = {
  clientId: import.meta.env.VITE_OAUTH_CLIENT_ID || 'your_client_id',
  redirectUri: import.meta.env.VITE_OAUTH_REDIRECT_URI || `${window.location.origin}/callback`,
  scopes: import.meta.env.VITE_OAUTH_SCOPES || 'openid profile email',
  authServerUrl: AUTH_SERVER_URL,
  apiBaseUrl: API_BASE_URL,
  endpoints: {
    authorize: '/oauth/authorize',
    userInfo: '/oauth/userinfo',
  },
}

/**
 * 인증 페이지 URL 생성 (브라우저 리다이렉트용)
 * 사용: window.location.href = getAuthPageUrl('authorize')
 */
export const getAuthPageUrl = (endpoint) => {
  return `${AUTH_CONFIG.authServerUrl}${AUTH_CONFIG.endpoints[endpoint]}`
}

/**
 * API 엔드포인트 URL 생성 (axios 호출용)
 * 개발 환경에서는 Vite 프록시를 통해 CORS 우회
 */
export const getApiUrl = (endpoint) => {
  return `${AUTH_CONFIG.apiBaseUrl}${AUTH_CONFIG.endpoints[endpoint]}`
}
```

### 4.2 authService.js - 인증 서비스

```javascript
/**
 * OAuth 인증 서비스
 * 인증 여부는 서버의 /oauth/userinfo API 호출로 확인
 */

import axios from 'axios'
import { AUTH_CONFIG, getAuthPageUrl, getApiUrl } from '../config/authConfig'

const USER_INFO_KEY = 'jabis_user_info'

/**
 * 인증 URL 생성
 */
export function getAuthUrl() {
  // CSRF 방지용 state 생성
  const state = crypto.randomUUID()
  sessionStorage.setItem('oauth_state', state)

  const params = new URLSearchParams({
    client_id: AUTH_CONFIG.clientId,
    redirect_uri: AUTH_CONFIG.redirectUri,
    response_type: 'code',
    scope: AUTH_CONFIG.scopes,
    state: state,
  })

  return `${getAuthPageUrl('authorize')}?${params}`
}

/**
 * 로그인 시작 (외부 인증 페이지로 리다이렉트)
 */
export function startLogin() {
  window.location.href = getAuthUrl()
}

/**
 * 콜백 처리
 */
export async function handleCallback(code, state) {
  // state 검증 (CSRF 방지)
  const savedState = sessionStorage.getItem('oauth_state')
  if (state && savedState && state !== savedState) {
    throw new Error('Invalid state parameter')
  }
  sessionStorage.removeItem('oauth_state')

  // 서버에 사용자 정보 요청
  try {
    const response = await axios.get(getApiUrl('userInfo'), {
      withCredentials: true,
    })

    const userInfo = response.data
    if (userInfo) {
      localStorage.setItem(USER_INFO_KEY, JSON.stringify(userInfo))
    }

    return userInfo
  } catch (error) {
    throw new Error('인증 토큰이 없습니다.')
  }
}

/**
 * 저장된 사용자 정보 가져오기
 */
export function getSavedUserInfo() {
  const userInfo = localStorage.getItem(USER_INFO_KEY)
  return userInfo ? JSON.parse(userInfo) : null
}

/**
 * 인증 여부 확인 (서버 검증)
 */
export async function verifyToken() {
  try {
    const response = await axios.get(getApiUrl('userInfo'), {
      withCredentials: true,
    })

    const userInfo = response.data
    if (userInfo) {
      localStorage.setItem(USER_INFO_KEY, JSON.stringify(userInfo))
    }

    return !!userInfo
  } catch (error) {
    return false
  }
}

/**
 * 로그아웃
 */
export function logout() {
  localStorage.removeItem(USER_INFO_KEY)
  localStorage.removeItem('jabis_role')
  sessionStorage.removeItem('oauth_state')
}

/**
 * Axios 인터셉터 설정
 */
export function setupAxiosInterceptors() {
  // 요청 인터셉터: withCredentials 설정
  axios.interceptors.request.use(
    (config) => {
      if (config.url?.includes('/auth-api') || config.url?.includes(AUTH_CONFIG.authServerUrl)) {
        config.withCredentials = true
      }
      return config
    },
    (error) => Promise.reject(error)
  )

  // 응답 인터셉터: 401 처리
  axios.interceptors.response.use(
    (response) => response,
    (error) => {
      if (error.response?.status === 401) {
        const currentPath = window.location.pathname
        const isAuthPage = currentPath === '/login' || currentPath.startsWith('/auth/')

        if (!isAuthPage) {
          logout()
          window.location.href = '/login'
        }
      }
      return Promise.reject(error)
    }
  )
}
```

### 4.3 authStore.js - Zustand 상태 관리

```javascript
/**
 * 인증 상태 관리 스토어
 */

import { create } from 'zustand'
import {
  getSavedUserInfo,
  logout as logoutService,
  startLogin,
  handleCallback as handleCallbackService,
  verifyToken,
} from '../services/authService'

export const useAuthStore = create((set, get) => ({
  isAuthenticated: false,
  user: getSavedUserInfo(),
  isLoading: false,
  error: null,

  login: () => {
    set({ isLoading: true, error: null })
    startLogin()
  },

  handleCallback: async (code, state) => {
    set({ isLoading: true, error: null })

    try {
      const userInfo = await handleCallbackService(code, state)

      set({
        isAuthenticated: true,
        user: userInfo,
        isLoading: false,
        error: null,
      })

      return userInfo
    } catch (error) {
      set({
        isAuthenticated: false,
        user: null,
        isLoading: false,
        error: error.message,
      })
      throw error
    }
  },

  logout: () => {
    logoutService()
    set({
      isAuthenticated: false,
      user: null,
      error: null,
    })
  },

  checkAuth: async () => {
    const isValid = await verifyToken()
    const user = getSavedUserInfo()

    set({
      isAuthenticated: isValid,
      user: isValid ? user : null,
    })

    return isValid
  },

  clearError: () => {
    set({ error: null })
  },
}))
```

---

## 5. 페이지 구현

### 5.1 LoginPage.jsx

```jsx
import React, { useEffect, useState } from 'react'
import { useNavigate } from 'react-router-dom'
import { useAuthStore } from '../stores/authStore'
import { verifyToken } from '../services/authService'

const OAUTH_ENABLED = import.meta.env.VITE_OAUTH_ENABLED === 'true'

export default function LoginPage() {
  const navigate = useNavigate()
  const { login, isLoading, error } = useAuthStore()
  const [isVerifying, setIsVerifying] = useState(OAUTH_ENABLED)

  // 이미 인증되어 있으면 리다이렉트
  useEffect(() => {
    const checkAuth = async () => {
      if (!OAUTH_ENABLED) {
        setIsVerifying(false)
        return
      }

      setIsVerifying(true)
      try {
        const isValid = await verifyToken()
        if (isValid) {
          navigate('/')
        }
      } catch (error) {
        console.error('Auth verification failed:', error)
      } finally {
        setIsVerifying(false)
      }
    }

    checkAuth()
  }, [navigate])

  if (isVerifying) {
    return <div>인증 확인 중...</div>
  }

  return (
    <div>
      {error && <div>{error}</div>}
      <button onClick={login} disabled={isLoading}>
        {isLoading ? '로그인 중...' : '통합 인증으로 로그인'}
      </button>
    </div>
  )
}
```

### 5.2 CallbackPage.jsx

```jsx
import React, { useEffect, useState } from 'react'
import { useNavigate, useSearchParams } from 'react-router-dom'
import { useAuthStore } from '../stores/authStore'
import { verifyToken } from '../services/authService'

export default function CallbackPage() {
  const navigate = useNavigate()
  const [searchParams] = useSearchParams()
  const [status, setStatus] = useState('processing')
  const [errorMessage, setErrorMessage] = useState('')

  const { handleCallback } = useAuthStore()

  useEffect(() => {
    const processCallback = async () => {
      const code = searchParams.get('code')
      const state = searchParams.get('state')
      const error = searchParams.get('error')
      const errorDescription = searchParams.get('error_description')

      // 에러 처리
      if (error) {
        setStatus('error')
        setErrorMessage(errorDescription || '인증에 실패했습니다.')
        setTimeout(() => navigate('/login'), 3000)
        return
      }

      // code가 없는 경우 - 이미 인증되어 있는지 확인
      if (!code) {
        const isValid = await verifyToken()
        if (isValid) {
          setStatus('success')
          setTimeout(() => navigate('/'), 500)
          return
        }
        setStatus('error')
        setErrorMessage('인증 코드가 없습니다.')
        setTimeout(() => navigate('/login'), 3000)
        return
      }

      try {
        await handleCallback(code, state)
        setStatus('success')
        setTimeout(() => navigate('/'), 1500)
      } catch (error) {
        setStatus('error')
        setErrorMessage(error.message)
        setTimeout(() => navigate('/login'), 3000)
      }
    }

    processCallback()
  }, [searchParams, handleCallback, navigate])

  return (
    <div>
      {status === 'processing' && <div>인증 처리 중...</div>}
      {status === 'success' && <div>로그인 성공!</div>}
      {status === 'error' && <div>{errorMessage}</div>}
    </div>
  )
}
```

### 5.3 인증이 필요한 페이지 (예: SelectRolePage.jsx)

```jsx
import React, { useEffect, useState } from 'react'
import { useNavigate } from 'react-router-dom'
import { verifyToken } from '../services/authService'

const OAUTH_ENABLED = import.meta.env.VITE_OAUTH_ENABLED === 'true'

export default function SelectRolePage() {
  const navigate = useNavigate()
  const [isVerifying, setIsVerifying] = useState(OAUTH_ENABLED)
  const [isAuthenticated, setIsAuthenticated] = useState(!OAUTH_ENABLED)

  useEffect(() => {
    const checkAuth = async () => {
      if (!OAUTH_ENABLED) {
        setIsAuthenticated(true)
        setIsVerifying(false)
        return
      }

      setIsVerifying(true)
      try {
        const isValid = await verifyToken()
        if (!isValid) {
          navigate('/login')
          return
        }
        setIsAuthenticated(true)
      } catch (error) {
        navigate('/login')
      } finally {
        setIsVerifying(false)
      }
    }

    checkAuth()
  }, [navigate])

  if (isVerifying) {
    return <div>인증 확인 중...</div>
  }

  if (!isAuthenticated) {
    return null
  }

  return (
    <div>
      {/* 페이지 내용 */}
    </div>
  )
}
```

---

## 6. Vite 프록시 설정 (CORS 해결)

개발 환경에서 인증 서버와 다른 포트를 사용할 경우 CORS 문제 해결을 위해 프록시 설정이 필요합니다.

### vite.config.js

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    proxy: {
      '/auth-api': {
        target: 'http://localhost:3000',  // 인증 서버 주소
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/auth-api/, '/api'),
        cookieDomainRewrite: '',
        secure: false,
        cookiePathRewrite: '/',
      },
    },
  },
})
```

### 프록시 동작 방식
```
브라우저 요청: /auth-api/oauth/userinfo
      ↓
Vite 프록시: rewrite → /api/oauth/userinfo
      ↓
인증 서버: http://localhost:3000/api/oauth/userinfo
```

---

## 7. App.jsx 라우팅 설정

```jsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom'
import LoginPage from './pages/LoginPage'
import CallbackPage from './pages/CallbackPage'
import SelectRolePage from './pages/SelectRolePage'

function App() {
  return (
    <BrowserRouter>
      <Routes>
        {/* 역할 선택 페이지 (루트) */}
        <Route path="/" element={<SelectRolePage />} />

        {/* 로그인 페이지 */}
        <Route path="/login" element={<LoginPage />} />

        {/* OAuth 콜백 */}
        <Route path="/auth/callback" element={<CallbackPage />} />

        {/* 기타 페이지들... */}

        {/* 404 */}
        <Route path="*" element={<Navigate to="/" replace />} />
      </Routes>
    </BrowserRouter>
  )
}

export default App
```

---

## 8. 저장소 사용 정리

| 저장소 | 키 | 용도 |
|--------|-----|------|
| sessionStorage | `oauth_state` | CSRF 방지용 state (로그인 시 생성, 콜백 시 검증 후 삭제) |
| localStorage | `jabis_user_info` | 사용자 정보 캐시 (서버에서 조회한 정보 저장) |
| localStorage | `jabis_role` | 선택한 역할 저장 |

---

## 9. 인증 플로우 상세

### 9.1 로그인 플로우
```
1. /login 페이지 접속
2. verifyToken() 호출 → 서버에 /oauth/userinfo 요청
3. 401 응답 (미인증) → 로그인 화면 표시
4. "통합 인증으로 로그인" 버튼 클릭
5. getAuthUrl() 호출 → state 생성 후 sessionStorage 저장
6. 인증 서버로 리다이렉트 (authorize 엔드포인트)
7. 사용자가 인증 서버에서 로그인
8. 인증 서버가 /auth/callback으로 리다이렉트 (code, state 포함)
9. CallbackPage에서 handleCallback(code, state) 호출
10. state 검증 (CSRF 방지)
11. 서버에 /oauth/userinfo 요청 → 사용자 정보 반환
12. 사용자 정보 localStorage에 저장
13. / (역할 선택 페이지)로 이동
```

### 9.2 인증 확인 플로우
```
1. 보호된 페이지 접속
2. verifyToken() 호출 → 서버에 /oauth/userinfo 요청 (withCredentials: true)
3. 서버가 쿠키 확인 후 사용자 정보 반환 또는 401
4. 200 응답 → 페이지 표시
5. 401 응답 → /login으로 리다이렉트
```

### 9.3 로그아웃 플로우
```
1. 로그아웃 버튼 클릭
2. logout() 호출
3. localStorage에서 사용자 정보, 역할 삭제
4. sessionStorage에서 oauth_state 삭제
5. /login으로 리다이렉트
```

---

## 10. 주의사항

### 10.1 CORS
- 개발 환경에서는 Vite 프록시 사용
- 운영 환경에서는 인증 서버에서 CORS 설정 또는 동일 도메인 사용

### 10.2 쿠키
- `withCredentials: true` 필수 (크로스 도메인 쿠키 전송)
- 인증 서버에서 쿠키 설정 시 `SameSite`, `Secure` 속성 확인

### 10.3 보안
- state 파라미터로 CSRF 공격 방지
- 인증 코드는 일회성으로 사용
- 토큰/세션은 서버에서 관리

### 10.4 OAuth 비활성화
- `VITE_OAUTH_ENABLED=false`로 설정하면 인증 검사 스킵
- 개발/테스트 시 유용
