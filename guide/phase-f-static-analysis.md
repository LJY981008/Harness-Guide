# Phase F — 정적분석 승격 (Spotless + Checkstyle)

> ← **[HARNESS_SETUP_GUIDE.md](../HARNESS_SETUP_GUIDE.md)** (상위 가이드로 돌아가기)
>
> **관련 Phase**: [A (Hooks)](phase-a-hooks.md) · [C (도메인 Skills)](phase-c-skills.md) · [G (자동 루프)](../HARNESS_SETUP_GUIDE.md#phase-g-자동-루프-posttooluse--stop-체인) · [H (주간 GC)](phase-h-weekly-gc.md)
>
> **역할**: 코딩 규칙을 **컴파일러 레벨에서 강제**. Phase A 훅의 유도 수준(`decision:block`) 위에 **빌드 실패**라는 진짜 강제 레일을 쌓는다. 위반 0건 달성 후 `ignoreFailures=false` 승격이 핵심.

## 목차
- [F-0. 전제 조건](#f-0-전제-조건)
- [F-1. build.gradle에 플러그인 추가](#f-1-buildgradle에-플러그인-추가)
- [F-2. Checkstyle 규칙 (최소 세트)](#f-2-checkstyle-규칙-최소-세트)
- [F-3. suppressions.xml](#f-3-suppressionsxml)
- [F-4. 포맷 베이스라인 적용 (1회성 대규모 커밋)](#f-4-포맷-베이스라인-적용-1회성-대규모-커밋)
- [F-5. wildcard import 자동 정리](#f-5-wildcard-import-자동-정리)
- [F-6. 엄격 모드 승격](#f-6-엄격-모드-승격)
- [F-7. 실측 결과](#f-7-실측-결과-7개-모듈-spring-boot-프로젝트-기준)

---

## Phase F: 정적분석 승격 (Spotless + Checkstyle)

> **목표**: CODING_RULES.md의 자동화 가능 규칙(wildcard import 금지, 네이밍, import 순서)을 Checkstyle로 **물리적 강제화**. [Phase A](phase-a-hooks.md)의 `decision:block`(유도)보다 강력.

### F-0. 전제 조건

- Java 21 + Gradle 8.x (Groovy DSL 또는 Kotlin DSL)
- 원격 브랜치 `origin/master`(또는 `origin/main`) 접근 가능

### F-1. build.gradle에 플러그인 추가

아래 예시는 Spring Boot 멀티모듈 Gradle 프로젝트 기준. 프로젝트 구조에 맞춰 `subprojects`/`allprojects` 블록 조정.

```groovy
plugins {
    // 기존 id들 + 아래 한 줄 추가
    id 'com.diffplug.spotless' version '8.4.0' apply false
}

subprojects {
    apply plugin: 'com.diffplug.spotless'
    apply plugin: 'checkstyle'

    spotless {
        ratchetFrom 'origin/master'  // 증분: 변경 파일만 검사
        java {
            target 'src/**/*.java'
            targetExclude 'build/generated/**/*'
            palantirJavaFormat('2.90.0')  // Lombok 친화
            removeUnusedImports()
            importOrder('java', 'javax', 'jakarta', 'org', 'com')
            formatAnnotations()
        }
    }

    checkstyle {
        toolVersion = '10.21.0'
        configFile = rootProject.file('config/checkstyle/checkstyle.xml')
        configProperties = [
            'suppressionFile': rootProject.file('config/checkstyle/suppressions.xml').toString()
        ]
        ignoreFailures = true   // 초기 도입: 리포트만, 빌드 실패 안 함
    }

    tasks.named('checkstyleTest') { enabled = false }

    // QueryDSL Q클래스가 main/test 공유 디렉토리 생성 시 implicit dep 해소
    tasks.named('checkstyleMain') {
        mustRunAfter(tasks.named('compileTestJava'))
    }
}
```

**버전 선정 근거**: 
- `palantir-java-format 2.90.0`: Java 21 공식 지원, Lombok `@Builder`/`@NoArgsConstructor` 호환
- `Spotless 8.4.0`: 최신 안정, Gradle 8.x 호환
- `Checkstyle 10.21.0`: Java 21 공식 지원
- **모두 실측 확인 버전** (추측 금지)

### F-2. Checkstyle 규칙 (최소 세트)

`config/checkstyle/checkstyle.xml`에 아래 최소 세트로 시작. 필요 시 점진 확장.

```xml
<module name="Checker">
    <property name="severity" value="error"/>
    <module name="SuppressionFilter">
        <property name="file" value="${suppressionFile}"/>
    </module>
    <module name="TreeWalker">
        <module name="AvoidStarImport"/>
        <module name="UnusedImports"><property name="processJavadoc" value="true"/></module>
        <module name="RedundantImport"/>
        <!-- Spotless importOrder와 정렬 일치 (CustomImportOrder는 그룹 분리 불일치로 오탐) -->
        <module name="ImportOrder">
            <property name="groups" value="/^java\./,/^javax\./,/^jakarta\./,/^org\./,/^com\./"/>
            <property name="ordered" value="true"/>
            <property name="separated" value="true"/>
            <property name="option" value="top"/>
            <property name="sortStaticImportsAlphabetically" value="true"/>
        </module>
        <module name="TypeName"/>
        <module name="MethodName"><property name="format" value="^[a-z][a-zA-Z0-9_]*$"/></module>
        <module name="ConstantName"/>
        <module name="PackageName"><property name="format" value="^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$"/></module>
        <module name="LocalVariableName"/>
        <module name="MemberName"/>
        <module name="ParameterName"/>
    </module>
</module>
```

**FQN 검출(Regexp) 제외 이유**: 실측 48건 대부분 `new java.util.concurrent.XXX<>()` 같은 정당한 사용. 오탐 비용 > 이익.

### F-3. suppressions.xml

```xml
<suppressions>
    <suppress files="[\\/]build[\\/]generated[\\/].*" checks=".*"/>
    <suppress files=".*[\\/]Q[A-Z]\w+\.java" checks=".*"/>  <!-- QueryDSL Q클래스 -->
    <suppress files=".*[\\/]src[\\/]test[\\/].*" checks=".*"/>
    <!-- QueryDSL static final Q*Entity 필드 재익스포트 관용 -->
    <suppress files=".*QueryDslRepository\.java" checks="ConstantName"/>
</suppressions>
```

### F-4. 포맷 베이스라인 적용 (1회성 대규모 커밋)

```bash
# 1. ratchetFrom 임시 주석 처리 (전체 파일 대상)
# 2. 일괄 apply
./gradlew spotlessApply --no-daemon

# 3. 변경 규모 확인 (대형 멀티모듈 기준 500+ 파일 / 1만+ 라인 예상)
git diff --stat | tail -3

# 4. 컴파일 검증 (포맷이 기능에 영향 없는지)
./gradlew compileJava --no-daemon

# 5. 별도 커밋 (blame 오염 격리)
git commit -am "chore: spotless palantir 포맷 베이스라인 적용"

# 6. .git-blame-ignore-revs에 커밋 SHA append
echo "$(git log -1 --format=%H)" >> .git-blame-ignore-revs
git commit -am "chore: 포맷 베이스라인 커밋을 blame-ignore-revs에 등록"

# 7. ratchetFrom 재활성화
# 8. 로컬 git 설정 (1회)
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

GitHub/GitLab 17.10+는 `.git-blame-ignore-revs` 자동 인식 — UI blame에서 해당 커밋 스킵.

### F-5. wildcard import 자동 정리

실측: 대형 프로젝트에서 80+ 파일의 wildcard import를 **실제 사용 심볼만의 명시 import**로 자동 전개 가능.

**핵심 설계**:
- 프로젝트 소스 + **JDK jmods** + Gradle 캐시 jar에서 각 wildcard 패키지의 public 클래스 추출
- 본문 대문자 식별자와 교집합 → 명시 import 생성
- **이미 명시 import된 심볼은 제외** (Component 같은 모호 오탐 방지)
- 실행 후 `./gradlew spotlessApply`로 정렬·포맷 보정

### F-6. 엄격 모드 승격

위반 0건 달성 후:

```groovy
checkstyle {
    // ...
    ignoreFailures = false   // 위반 1건이라도 빌드 차단
    maxWarnings = 0
}
```

이제 `./gradlew build -x test`가 Checkstyle 위반 시 실패 → [Phase A](phase-a-hooks.md)의 post-work-check가 block 반환 → 실질적 강제.

### F-7. 실측 결과 (7개 모듈 Spring Boot 프로젝트 기준)

| 항목 | Before | After |
|---|---|---|
| wildcard import | 81 파일 / 280+ 라인 | **0** |
| Checkstyle 위반 | 측정 불가 (규칙 없음) → 초기 1,347건 → 104건 (ImportOrder 정렬 교체 후) → 42건 (Spotless 적용 후) → **0** |
| ignoreFailures | n/a | `true` → `false` |
| 포맷 일관성 | 혼재 | palantir 2.90.0 통일 |
| 베이스라인 커밋 | n/a | 507 파일 (SHA는 .git-blame-ignore-revs) |
