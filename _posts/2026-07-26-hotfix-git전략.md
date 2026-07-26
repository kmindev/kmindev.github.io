---
title: "긴급 버그 발생 시 hotfix 브랜치 전략"
date: 2026-07-26 00:00:00 +0900
categories: [Git]
tags: [Git, Branch, Hotfix, 배포전략]
---

## 배경

현재 프로젝트는 한 달에 한 번 정기배포를 하고, 긴급 버그가 발생하면 hotfix로 나간다.

배포 후 운영상황에 돌입하다보니 한가지 고민이 생겼다.

예를 들어, 7월 정기배포가 끝나고, 8월 정기배포를 위한 기능들을 develop에 붙여가던 중 갑자기 긴급 버그가 발생해 hotfix를 배포해야 하는 상황이 오면, 
develop에는 이미 8월 기능이 올라가 있는 상태라, 그냥 develop을 배포하면 아직 나가면 안 되는 기능까지 같이 나가버린다.

## 해결 방법

해결 방법은 여러 가지 있겠지만, 가장 간단한 방법이다.

main 브랜치에서 hotfix 브랜치를 따서 수정 후 main에 머지하고 배포한다. 
main은 7월 배포 이후 상태 그대로이기 때문에 8월 기능이 포함되지 않는다.

```
main → hotfix/xxx → (수정) → main 머지 → hotfix 배포
이후 main → develop 역머지 (싱크)
```

역머지를 빠뜨리면 main에만 hotfix 커밋이 존재하고 develop에는 없는 상태가 되어, 나중에 머지할 때 충돌이 나거나 hotfix가 유실될 수 있으므로 반드시 챙겨야 한다.

## 명령어

```bash
# main에서 hotfix 브랜치 생성
git checkout main
git pull origin main
git checkout -b hotfix/xxx

# 수정 후 커밋
git add .
git commit -m "fix: xxx"

# main에 머지
git checkout main
git merge hotfix/xxx
git push origin main

# develop 역머지 (싱크)
git checkout develop
git merge main
git push origin develop

```
