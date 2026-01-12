# wordwolf-lobby2

4인용 워드울프(라이어) 웹앱 MVP 스펙이랑 초기 구조 정리해뒀어.

## 1. 간단하게 스택 고르기

| 영역 | 선택지 | 이유 |
| --- | --- | --- |
| **클라이언트** | **React(Vite) + Tailwind** | 가볍고 모바일 반응형 작업 편함 |
| **실시간 통신** | **socket.io** | Replit에서도 바로 돌아가는 WebSocket 레이어 |
| **서버** | **Node(Express) + socket.io** | 별도 인프라 없이 Replit 한 곳에서 실행 |
| **상태 관리** | **Zustand** | 프런트-엔드 글로벌 상태 간단하게 관리 |
| **룸 코드 생성** | **nanoid** | 충돌 거의 없는 짧은 랜덤 코드 |
| **배포** | **Replit**(완성 → GitHub → Replit Deploy) or **Vercel**(Next.js 쓴다면) | 무료 + 편리한 CI/CD |

## 2. 최소 기능 정의 (MVP)

1. **방 만들기**
   - `POST /room` → `{ roomCode, hostId }`
   - 방 정보는 서버 메모리에만 보관 (작아서 OK).
2. **방 참여**
   - 클라이언트가 `socket.emit('join', { roomCode, userId })`.
   - 서버는 해당 `roomCode`가 있으면 `socket.join(roomCode)`.
3. **대기실(Lobby)**
   - 서버가 현재 인원 리스트를 `room:update` 이벤트로 브로드캐스트.
4. **게임 시작 (호스트만)**
   - `socket.emit('start', { roomCode })`.
   - 서버는
     1. 참가자 5명인지 확인
     2. 미리 준비한 `topics.json`에서 주제-제시어 뽑기
     3. 각 유저에게 - `prompt: word` / 라이어에겐 `prompt: null + topic`.
5. **게임 종료 & 재시작**
   - 프런트에서 “다시 하기” 누르면 `socket.emit('restart')`, 서버는 같은 방에서 4번 로직 재실행.

## 3. 폴더 구조 예시

```
liargame/
├─ server/
│  └─ index.js          # Express + socket.io
├─ client/
│  ├─ src/
│  │  ├─ pages/
│  │  │  ├─ Home.tsx   # 방 생성 / 입장
│  │  │  ├─ Lobby.tsx  # 대기실
│  │  │  └─ Game.tsx   # 제시어 보여주기
│  │  ├─ store.ts      # Zustand
│  │  └─ sockets.ts    # socket.io client
│  └─ vite.config.ts
├─ topics.json          # { "음식": ["김치", "초밥", ...], ... }
├─ package.json         # 루트에서 workspace로 client/server 둘 다 관리
└─ README.md
```

> Replit 사용 시 **“Node.js (with Express)”** 템플릿으로 만들고, `client`는 Vite dev 서버를 프록시하도록 설정하면 돼.
> Vercel 배포를 원하면 Next.js 단일 리포로 합쳐도 OK. (API Routes = WebSocket 불가 → `socket.io` 서버 별도 필요)

## 4. 서버 구현 핵심 (server/index.js)

```js
import express from 'express';
import { createServer } from 'http';
import { Server } from 'socket.io';
import { nanoid } from 'nanoid';
import topics from '../topics.json' assert { type: 'json' };

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, { cors: { origin: '*' } });

const rooms = new Map(); // roomCode -> { hostId, players: Map }

app.use(express.json());

// REST: 방 생성
app.post('/room', (req, res) => {
  const roomCode = nanoid(6).toUpperCase();
  rooms.set(roomCode, { hostId: null, players: new Map() });
  res.json({ roomCode });
});

// Socket: 참가 & 이벤트
io.on('connection', (socket) => {
  socket.on('join', ({ roomCode, userId, nick }) => {
    const room = rooms.get(roomCode);
    if (!room) return socket.emit('error', '방 없음');
    if (!room.hostId) room.hostId = socket.id;

    room.players.set(socket.id, { userId, nick });
    socket.join(roomCode);

    io.to(roomCode).emit('room:update', Array.from(room.players.values()));
  });

  socket.on('start', ({ roomCode }) => {
    const room = rooms.get(roomCode);
    if (!room || socket.id !== room.hostId) return;

    if (room.players.size !== 5) return socket.emit('error', '5명 필요');

    // 제시어 뿌리기
    const [topic, words] = randomTopic();
    const liarIndex = Math.floor(Math.random() * 5);

    [...room.players.keys()].forEach((id, idx) => {
      io.to(id).emit('prompt', {
        topic,
        word: idx === liarIndex ? null : words[idx],
        isLiar: idx === liarIndex,
      });
    });
  });

  socket.on('disconnect', () => {
    // 방에서 제거 & 업데이트
    for (const [code, room] of rooms) {
      if (room.players.delete(socket.id)) {
        io.to(code).emit('room:update', Array.from(room.players.values()));
        if (room.players.size === 0) rooms.delete(code);
      }
    }
  });
});

function randomTopic() {
  const keys = Object.keys(topics);
  const topic = keys[Math.floor(Math.random() * keys.length)];
  const words = shuffle([...topics[topic]]).slice(0, 5);
  return [topic, words];
}

function shuffle(arr) { return arr.sort(() => Math.random() - 0.5); }

httpServer.listen(3000);
```

*위 코드는 이해를 돕기 위한 스케치. 예외 처리·메모리 누수·보안은 추가해야 함.*

## 5. 클라이언트 주요 로직

```tsx
// sockets.ts
import { io } from 'socket.io-client';
export const socket = io(import.meta.env.VITE_SOCKET_URL);

// store.ts
import { create } from 'zustand';
export const useStore = create(set => ({
  players: [], topic: '', word: '', isLiar: false,
  setPlayers: (p) => set({ players: p }),
  setPrompt: (topic, word, isLiar) => set({ topic, word, isLiar }),
}));

// Lobby.tsx (예시)
useEffect(() => {
  socket.on('room:update', setPlayers);
  socket.on('prompt', ({ topic, word, isLiar }) =>
    setPrompt(topic, word, isLiar)
  );
  return () => socket.offAny();
}, []);
```

## 6. 모바일 UI 빠르게 다듬는 팁

- **Tailwind**로 `max-w-[500px] mx-auto p-4` 래퍼 하나 두고 내부 컴포넌트 세로 정렬.
- Panic 버튼(나가기) 고정 `fixed bottom-4 right-4`.
- 뷰포트 길이 이슈 해결: `h-[100dvh]` 활용.
- 리액트 라우터 대신 Zustand + 조건부 렌더로 단계(state) 관리하면 번거로움 ↓.

## 7. GitHub & 배포

1. **Git 초기화**

   ```bash
   git init && gh repo create liargame --public -y
   git add . && git commit -m "init"
   git push -u origin main
   ```

2. **Replit**

   - 새 Repl → “Import from GitHub” → URL 입력 → 자동 빌드.
   - Secrets 탭에 `VITE_SOCKET_URL` 같은 변수 설정.

3. **Vercel (선택)**

   - `vercel init` → Framework = **Other**.
   - `vercel.json`에:

     ```json
     { "functions": { "api/index.js": { "runtime": "nodejs18.x" } } }
     ```

   - 서버 코드 `api/index.js`, 클라이언트는 `public/`으로 빌드 후 업로드.

## 8. 다음 단계 (고도화 아이디어)

- 방 코드 만료 & Redis 같은 외부 캐시 연결.
- 라이어 투표 UI + 결과 집계.
- 구글 OAuth 로그인으로 유저 식별.
- PWA 설정(서비스 워커) → 오프라인 스플래시 화면.

## 정리

`React + socket.io + Node(Express)` 조합이면 Replit 하나로 **방 생성 → 실시간 게임 → 재시작**까지 충분히 구현 가능해. 위 구조 그대로 따라가면서 필요 기능 붙이면 바로 GitHub 푸시 후 Replit Deploy 땡. 중간에 막히는 부분 생기면 구체적으로 질문해줘!
