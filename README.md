# Korean Game Localization Skills

레트로 콘솔·PC 게임 한글화 과정에서 검증한 구조 분석과 작업 지식을 Codex 스킬로 공개합니다.

## 공개 스킬

- `game-dialogue-expansion`: 번역문 길이가 달라질 때 길이 필드, 누적 오프셋, 절대 포인터, 중첩 포인터와 컨테이너 구조를 안전하게 재계산합니다.
  - Type 1~5: 검증된 구조
  - Type 6: 나의 여름방학1 기반 잠정 구조이며 개선사항이 명시되어 있습니다.

향후 파일 구조 분석, 텍스트 추출, 폰트·인코딩, 재삽입 및 검증 스킬을 별도로 추가합니다.

## Codex에서 설치

Codex에 다음과 같이 요청하세요.

```text
$skill-installer를 사용해 다음 GitHub 스킬을 설치해 주세요:
https://github.com/MINGS-KO-PATCH/korean-game-localization-skills/tree/main/skills/game-dialogue-expansion
```

명령줄 설치 도구를 사용하는 경우:

```text
install-skill-from-github.py --repo MINGS-KO-PATCH/korean-game-localization-skills --path skills/game-dialogue-expansion
```

설치 후 다음 대화부터 `$game-dialogue-expansion`을 사용할 수 있습니다.

## 변경 제안과 권한

저장소는 공개이므로 누구나 읽기, 설치, 포크, 변경 제안(PR)을 할 수 있습니다. 원본 저장소에 직접 쓰는 권한은 소유자 또는 소유자가 명시적으로 권한을 부여한 협업자에게만 있습니다. PR은 소유자가 검토하고 병합해야 원본에 반영됩니다.

## 자료 원칙

- 특정 게임과 리비전에서 확인한 수치는 사례 근거로 기록합니다.
- 다른 게임에 적용할 때는 포인터 기준점, 폭, 엔디언, 경계와 소비 경로를 다시 검증합니다.
- 원본 ROM·디스크 이미지와 권한 없는 제3자 자산은 포함하지 않습니다.
