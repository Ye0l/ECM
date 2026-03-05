# sv

Everything you need to build a Svelte project, powered by [`sv`](https://github.com/sveltejs/cli).

## Creating a project

If you're seeing this, you've probably already done this step. Congrats!

```sh
# create a new project
npx sv create my-app
```

To recreate this project with the same configuration:

```sh
# recreate this project
npx sv@0.12.5 create --template minimal --types ts --install npm ECM
```

## Developing

Once you've created a project and installed dependencies with `npm install` (or `pnpm install` or `yarn`), start a development server:

```sh
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To create a production version of your app:

```sh
npm run build
```

You can preview the production build with `npm run preview`.


> To deploy your app, you may need to install an [adapter](https://svelte.dev/docs/kit/adapters) for your target environment.

## 변경 사항

### 2026-03-05
- **.gitignore 보강**: SvelteKit 프로젝트에 적합하도록 포괄적인 `.gitignore` 파일 설정을 추가했습니다. (의존성, 빌드 결과물, IDE 설정, 환경 변수 등)
- **복식부기 및 물리 엔진 가계부(V2)**: `d3-force` 모듈을 도입해 캔버스 내 노드 간 인력/척력을 구현하였습니다. `NodeEditor`로 다중 차/대변(n:n) 거래를 입력할 수 있으며, 툴바 검색창을 통해 동적인 중앙 앵커 필터 노드를 중심으로 관련 항목이 자동 결합되는 레이아웃을 구현했습니다.
