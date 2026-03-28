# C++ RPG Game — Learning Project

## 프로젝트 목적

이 프로젝트는 **게임 개발 자체가 목적이 아닙니다.**
C++ 핵심 소프트웨어 엔지니어링 개념을 실제 코드로 체득하는 것이 목적입니다.

학습 목표 세 가지:
1. **SOLID 원칙** — SRP, OCP, LSP, ISP, DIP 를 코드로 직접 표현
2. **Clean Architecture** — 레이어 분리, 의존성 방향 규칙 준수
3. **Smart Pointer** — `unique_ptr`, `shared_ptr`, `weak_ptr`, 커스텀 구현

> 코드를 작성하거나 리뷰할 때 항상 이 세 가지 렌즈로 먼저 바라볼 것.

---

## 아키텍처 — Clean Architecture 레이어

의존성은 반드시 **바깥 → 안쪽** 방향으로만 흐릅니다.
안쪽 레이어는 바깥 레이어를 절대 import/include 하지 않습니다.

```
┌──────────────────────────────────────────────┐
│  Frameworks & Drivers  (4번째 레이어 — 최외곽) │
│  src/frameworks/                              │
│  - ConsolePresenter.cpp                       │
│  - FileSystem.cpp                             │
│  - 외부 라이브러리 어댑터                       │
├──────────────────────────────────────────────┤
│  Interface Adapters  (3번째 레이어)            │
│  src/adapters/                                │
│  - JsonCharacterRepository.cpp                │
│  - BattleController.cpp                       │
│  - 내부 ↔ 외부 데이터 변환 담당               │
├──────────────────────────────────────────────┤
│  Use Cases  (2번째 레이어)                     │
│  src/usecases/                                │
│  - ExecuteBattleUseCase.cpp                   │
│  - SaveGameUseCase.cpp                        │
│  - 순수 비즈니스 로직만, UI·DB 접근 없음        │
├──────────────────────────────────────────────┤
│  Entities  (1번째 레이어 — 핵심)               │
│  src/entities/                                │
│  - Character.cpp, Item.cpp                    │
│  - Skill.cpp, Quest.cpp                       │
│  - 외부 의존성 제로, 순수 도메인               │
└──────────────────────────────────────────────┘
```

### 의존성 규칙 위반 예시 (절대 금지)

```cpp
// ❌ Entity가 파일 I/O를 직접 아는 것 — 레이어 위반
class Character {
    void save() { std::ofstream f("save.json"); } // 금지
};

// ✅ Repository 인터페이스를 통해 역전
class ICharacterRepository {
public:
    virtual void save(const Character& c) = 0;
};
```

---

## SOLID 원칙 — 원칙별 코드 규칙

### SRP — 단일 책임 원칙 (Phase 1)

클래스 하나 = 변경 이유 하나.

```cpp
// ❌ 나쁜 예 — Character가 너무 많이 안다
class Character {
    void attack() { ... }
    void render() { ... }       // 렌더링은 다른 책임
    void saveToFile() { ... }   // 저장은 다른 책임
};

// ✅ 좋은 예 — 책임 분리
class Character       { /* 데이터 + 도메인 로직만 */ };
class CharacterStats  { /* 스탯 계산 전담 */ };
class CharacterRenderer { /* 출력 전담 */ };
```

### OCP — 개방/폐쇄 원칙 (Phase 1)

확장에는 열려있고, 수정에는 닫혀있다. 새 직업 추가 시 기존 코드 수정 금지.

```cpp
// 인터페이스 — 절대 수정하지 않는 계약
class ICharacterClass {
public:
    virtual ~ICharacterClass() = default;
    virtual std::string getName() const = 0;
    virtual int getBaseAttack() const = 0;
    virtual int getBaseDefense() const = 0;
    virtual std::unique_ptr<ISkill> getSignatureSkill() const = 0;
};

// 확장 — 기존 코드 건드리지 않고 새 직업 추가
class Warrior : public ICharacterClass { ... };
class Mage    : public ICharacterClass { ... };
class Archer  : public ICharacterClass { ... };
// 새 직업: class Paladin : public ICharacterClass { ... }; ← 추가만 하면 됨
```

### LSP — 리스코프 치환 원칙 (Phase 2)

기반 타입 자리에 파생 타입을 넣어도 프로그램이 올바르게 동작해야 한다.

```cpp
// ✅ LSP 준수 — 모든 파생 클래스가 계약을 지킴
class IAttackStrategy {
public:
    // 사전조건: damage > 0
    // 사후조건: 반환값은 실제 적용된 피해량 (0 이상)
    virtual int execute(Character& attacker, Character& target) = 0;
};

// ❌ LSP 위반 예시
class BrokenSkill : public IAttackStrategy {
    int execute(Character& attacker, Character& target) override {
        return -999; // 사후조건 위반 — 음수 반환
    }
};
```

### ISP — 인터페이스 분리 원칙 (Phase 3)

클라이언트는 자신이 사용하지 않는 메서드에 의존하지 않아야 한다.

```cpp
// ❌ 비대한 인터페이스 — 금지
class IRepository {
    virtual Character loadCharacter(int id) = 0;
    virtual void saveCharacter(const Character& c) = 0;
    virtual Item loadItem(int id) = 0;
    virtual void saveItem(const Item& i) = 0;
    virtual Quest loadQuest(int id) = 0;
    // ...읽기만 필요한 클라이언트도 저장 메서드를 구현해야 함
};

// ✅ 분리된 인터페이스 — 필요한 것만 의존
class ICharacterReader { virtual Character load(int id) = 0; };
class ICharacterWriter { virtual void save(const Character& c) = 0; };
class IItemRepository  { virtual Item loadItem(int id) = 0; virtual void saveItem(const Item&) = 0; };
```

### DIP — 의존성 역전 원칙 (Phase 2~4)

고수준 모듈이 저수준 모듈에 의존하지 않는다. 둘 다 추상에 의존한다.

```cpp
// ❌ 고수준이 저수준(구체 클래스)에 의존 — 금지
class BattleSystem {
    PhysicalAttack attack; // 구체 클래스 직접 사용
};

// ✅ 추상(인터페이스)에 의존
class BattleSystem {
    std::shared_ptr<IAttackStrategy> attackStrategy; // 인터페이스 의존
public:
    explicit BattleSystem(std::shared_ptr<IAttackStrategy> strategy)
        : attackStrategy(std::move(strategy)) {}
};
```

---

## Smart Pointer — 사용 규칙

### 원칙: `new` / `delete` 직접 사용 금지

모든 동적 할당은 smart pointer 또는 컨테이너로 관리합니다.

### unique_ptr — 단독 소유권 (Phase 1)

```cpp
// 생성: make_unique 사용
auto warrior = std::make_unique<Warrior>();

// 소유권 이전: std::move
std::unique_ptr<ICharacterClass> cls = std::move(warrior);

// 함수 인자: 소유권 이전 시 값, 빌림 시 raw pointer 또는 reference
void takeOwnership(std::unique_ptr<Character> c);   // 소유권 가져감
void borrow(const Character& c);                    // 빌리기만 함
void borrow(Character* c);                          // nullable 빌리기

// ❌ unique_ptr 복사 금지
auto a = std::make_unique<Character>();
auto b = a; // 컴파일 에러 — 의도한 것
```

### shared_ptr — 공유 소유권 (Phase 2)

```cpp
// 생성: make_shared 사용 (new 직접 사용 금지)
auto hero = std::make_shared<Character>("Hero");

// 여러 시스템에서 공유
battleSystem->addParticipant(hero);
uiSystem->setFocus(hero);
// hero의 실제 소멸은 마지막 참조가 사라질 때

// ⚠️ 순환 참조 주의
class Party {
    std::vector<std::shared_ptr<Character>> members;
};
class Character {
    // std::shared_ptr<Party> party; // ❌ 순환 참조 발생
    std::weak_ptr<Party> party;      // ✅ weak_ptr로 해결
};
```

### weak_ptr — 비소유 관찰 (Phase 3)

```cpp
// weak_ptr 사용 패턴
class EventListener {
    std::weak_ptr<Character> target;
public:
    void onEvent() {
        if (auto t = target.lock()) { // lock()으로 유효성 확인
            t->notify();
        }
        // 이미 소멸된 경우 lock()은 nullptr 반환 — 안전
    }
};
```

### 커스텀 SharedPtr (Phase 4 목표)

```cpp
// 직접 구현할 레퍼런스 카운팅 스마트 포인터
template<typename T>
class SharedPtr {
    T* ptr;
    int* refCount;
public:
    explicit SharedPtr(T* p = nullptr);
    SharedPtr(const SharedPtr& other);   // 복사: refCount++
    SharedPtr(SharedPtr&& other);        // 이동: 소유권 이전
    ~SharedPtr();                        // refCount-- , 0이면 delete
    SharedPtr& operator=(const SharedPtr&);
    T& operator*() const;
    T* operator->() const;
    int use_count() const;
};
```

---

## 프로젝트 디렉토리 구조

```
rpg-game/
├── CLAUDE.md                    ← 이 파일
├── CMakeLists.txt
├── README.md
│
├── src/
│   ├── entities/                ← 레이어 1: 도메인 (외부 의존성 없음)
│   │   ├── Character.hpp / .cpp
│   │   ├── CharacterStats.hpp / .cpp
│   │   ├── Item.hpp / .cpp
│   │   ├── Skill.hpp / .cpp
│   │   └── Quest.hpp / .cpp
│   │
│   ├── usecases/                ← 레이어 2: 비즈니스 로직
│   │   ├── interfaces/
│   │   │   ├── ICharacterRepository.hpp
│   │   │   ├── IAttackStrategy.hpp
│   │   │   └── IPresenter.hpp
│   │   ├── ExecuteBattleUseCase.hpp / .cpp
│   │   ├── SaveGameUseCase.hpp / .cpp
│   │   └── LoadGameUseCase.hpp / .cpp
│   │
│   ├── adapters/                ← 레이어 3: 인터페이스 어댑터
│   │   ├── JsonCharacterRepository.hpp / .cpp
│   │   ├── BattleController.hpp / .cpp
│   │   └── GamePresenter.hpp / .cpp
│   │
│   ├── frameworks/              ← 레이어 4: 외부 (UI, 파일, 라이브러리)
│   │   ├── ConsoleUI.hpp / .cpp
│   │   └── FileSystem.hpp / .cpp
│   │
│   └── main.cpp                 ← DI 컨테이너, 최상위 조립
│
└── tests/
    ├── entities/
    ├── usecases/
    └── adapters/
```

---

## 8주 개발 계획

### Phase 1 — 기반 설계 & 엔티티 시스템 (1~2주차)

**학습 포인트:** SRP, OCP, `unique_ptr`, Entity 레이어

**1주차 태스크:**
- [ ] `ICharacterClass` 인터페이스 정의
- [ ] `Warrior`, `Mage`, `Archer` 구현 (OCP)
- [ ] `CharacterStats` 분리 (SRP)
- [ ] `CharacterRenderer` 분리 (SRP)
- [ ] 각 클래스 단위 테스트 작성

**2주차 태스크:**
- [ ] `CharacterFactory` — `unique_ptr` 반환, OCP 적용
- [ ] `unique_ptr` 소유권 이전 데모 (move semantics)
- [ ] Entity 레이어 레이어 위반 여부 검토 (외부 include 없어야 함)
- [ ] 콘솔 데모: 직업 선택 → 캐릭터 생성 → 스탯 출력

**핵심 인터페이스:**
```cpp
class ICharacterClass {
public:
    virtual ~ICharacterClass() = default;
    virtual std::string getName() const = 0;
    virtual int getBaseAttack() const = 0;
    virtual int getBaseDefense() const = 0;
    virtual int getBaseHP() const = 0;
    virtual int getBaseMP() const = 0;
    virtual std::unique_ptr<ISkill> getSignatureSkill() const = 0;
};
```

---

### Phase 2 — 전투 시스템 & 의존성 역전 (3~4주차)

**학습 포인트:** LSP, DIP, `shared_ptr`, Use Case 레이어

**3주차 태스크:**
- [ ] `IAttackStrategy` 인터페이스 정의
- [ ] `PhysicalAttack`, `MagicAttack`, `StatusEffectAttack` 구현
- [ ] LSP 검증: 모든 구현체가 사전/사후 조건 준수하는지 테스트
- [ ] `shared_ptr<Character>` 로 전투 참가자 공유

**4주차 태스크:**
- [ ] `ExecuteBattleUseCase` 구현 (DIP: 인터페이스에만 의존)
- [ ] 턴제 전투 루프 (선공 결정 → 행동 선택 → 피해 계산 → 상태 갱신)
- [ ] 피해 공식: `damage = max(1, attack - defense/2)`
- [ ] 전투 결과를 `IPresenter`로 출력 (구체 UI 모름)
- [ ] 콘솔 데모: 플레이어 vs 몬스터 전투 시뮬레이션

**핵심 인터페이스:**
```cpp
class IAttackStrategy {
public:
    virtual ~IAttackStrategy() = default;
    // 사후조건: 반환값 >= 0
    virtual int execute(Character& attacker, Character& target) = 0;
    virtual std::string getDescription() const = 0;
};

class ExecuteBattleUseCase {
    std::shared_ptr<IAttackStrategy> strategy;
    std::shared_ptr<IPresenter> presenter;
public:
    ExecuteBattleUseCase(
        std::shared_ptr<IAttackStrategy> s,
        std::shared_ptr<IPresenter> p
    );
    BattleResult execute(Character& player, Character& enemy);
};
```

---

### Phase 3 — 데이터 영속성 & 인터페이스 분리 (5~6주차)

**학습 포인트:** ISP, `weak_ptr`, Interface Adapter 레이어, Repository 패턴

**5주차 태스크:**
- [ ] `ICharacterReader`, `ICharacterWriter` 분리 정의 (ISP)
- [ ] `IItemRepository`, `IQuestRepository` 정의
- [ ] `JsonCharacterRepository` 구현 (파일 저장/로드)
- [ ] JSON 직렬화: `nlohmann/json` 또는 직접 구현
- [ ] Repository 교체 테스트: `InMemoryCharacterRepository` 로 대체 확인

**6주차 태스크:**
- [ ] 인벤토리 시스템 (`Character`가 `Item` 목록 보유)
- [ ] 파티 시스템 구현
- [ ] `weak_ptr` 순환 참조 방지 적용 (Party ↔ Character)
- [ ] Observer 패턴에서 `weak_ptr` 활용 (이벤트 리스너)
- [ ] 세이브/로드 콘솔 데모

**핵심 인터페이스:**
```cpp
// ISP 적용 — 읽기/쓰기 분리
class ICharacterReader {
public:
    virtual ~ICharacterReader() = default;
    virtual std::optional<Character> findById(int id) const = 0;
    virtual std::vector<Character> findAll() const = 0;
};

class ICharacterWriter {
public:
    virtual ~ICharacterWriter() = default;
    virtual void save(const Character& character) = 0;
    virtual void remove(int id) = 0;
};

// 순환 참조 방지 패턴
class Party {
    std::vector<std::shared_ptr<Character>> members;
};
class Character {
    std::weak_ptr<Party> party; // shared_ptr이면 순환 → weak_ptr
};
```

---

### Phase 4 — 맵 & 이벤트 시스템, 전체 통합 (7~8주차)

**학습 포인트:** 전체 SOLID 통합, 커스텀 `SharedPtr<T>`, DI 컨테이너, Presenter 레이어

**7주차 태스크:**
- [ ] 월드맵 — 타일 기반, 이동 가능 여부 판단
- [ ] NPC 대화 시스템 — 대화 트리 구조
- [ ] 퀘스트 시스템 — 수락/진행/완료 상태 관리
- [ ] `IPresenter` 구현체: `ConsolePresenter`
- [ ] 전체 게임 루프 연결 (맵 이동 → 전투 → 세이브)

**8주차 태스크:**
- [ ] 커스텀 `SharedPtr<T>` 구현 (레퍼런스 카운팅)
- [ ] `std::shared_ptr` 과 동작 비교 테스트
- [ ] 간단한 DI 컨테이너 구현
- [ ] 전체 SOLID 위반 코드 리뷰 & 리팩터링
- [ ] 플레이어블 데모 완성

**DI 컨테이너 스케치:**
```cpp
class DIContainer {
    std::unordered_map<std::string, std::function<std::shared_ptr<void>()>> factories;
public:
    template<typename T>
    void bind(std::function<std::shared_ptr<T>()> factory) {
        factories[typeid(T).name()] = [factory]() {
            return std::static_pointer_cast<void>(factory());
        };
    }

    template<typename T>
    std::shared_ptr<T> resolve() {
        return std::static_pointer_cast<T>(factories[typeid(T).name()]());
    }
};

// main.cpp에서 조립
DIContainer container;
container.bind<ICharacterRepository>([]() {
    return std::make_shared<JsonCharacterRepository>("save.json");
});
container.bind<IAttackStrategy>([]() {
    return std::make_shared<PhysicalAttack>();
});
auto useCase = std::make_shared<ExecuteBattleUseCase>(
    container.resolve<IAttackStrategy>(),
    container.resolve<IPresenter>()
);
```

---

## 코드 작성 규칙

### 필수 규칙
- `new` / `delete` 직접 사용 **금지** — `make_unique`, `make_shared` 사용
- 모든 인터페이스는 `virtual ~Interface() = default;` 포함
- 구체 클래스 직접 include 시 레이어 방향 반드시 확인
- 함수 하나는 20줄 이내 목표 (SRP)
- 새 기능 추가 전 인터페이스 먼저 정의 (DIP, OCP)

### 네이밍 컨벤션
- 인터페이스: `I` 접두사 (예: `ICharacterClass`, `IAttackStrategy`)
- Use Case: `~UseCase` 접미사 (예: `ExecuteBattleUseCase`)
- Repository: `~Repository` 접미사 (예: `JsonCharacterRepository`)
- 멤버 변수: `m_` 접두사 또는 trailing `_` (팀 통일)

### 테스트
- 모든 Entity, Use Case는 단위 테스트 작성
- Repository는 `InMemory` 구현체로 테스트 (파일 I/O 없이)
- LSP 검증: 기반 타입으로 모든 파생 타입을 교체하는 테스트 케이스

---

## 자주 쓰는 명령어 (Claude Code 세션 내)

```
# 현재 Phase 확인
"현재 Phase 1 2주차야. CharacterFactory 구현 도와줘."

# 레이어 위반 검토
"src/entities/ 디렉토리 파일들이 외부 레이어를 include하고 있는지 확인해줘."

# SOLID 위반 검토
"Character.cpp 읽고 SRP 위반 여부 분석해줘."

# 테스트 작성
"ExecuteBattleUseCase 에 대한 단위 테스트 작성해줘. Repository는 InMemory mock 써."

# 다음 태스크
"Phase 2 3주차 태스크 목록 보여주고 첫 번째부터 시작하자."
```

---

## 현재 진행 상황

**현재 Phase:** Phase 1 — 1주차  
**완료 태스크:** 없음 (시작 전)  
**다음 태스크:** `ICharacterClass` 인터페이스 정의 및 `Warrior`, `Mage`, `Archer` 구현

> 태스크 완료 시 이 파일의 체크박스(`- [ ]` → `- [x]`)를 업데이트하여 진행 상황 추적