# Detekt Setup

## Overview
Detekt is a static code analysis tool for Kotlin that helps you write cleaner code and detect code smells.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
detekt = "1.23.7"

[plugins]
detekt = { id = "io.gitlab.arturbosch.detekt", version.ref = "detekt" }
```

### 2. Apply Plugin

In root `build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.detekt)
}

detekt {
    buildUponDefaultConfig = true
    allRules = false
    config.setFrom(files("$projectDir/config/detekt/detekt.yml"))
    baseline = file("$projectDir/config/detekt/baseline.xml")
    
    reports {
        html.required.set(true)
        xml.required.set(true)
        txt.required.set(true)
        sarif.required.set(true)
        md.required.set(true)
    }
}

dependencies {
    detektPlugins("io.gitlab.arturbosch.detekt:detekt-formatting:1.23.7")
}

tasks.withType<io.gitlab.arturbosch.detekt.Detekt>().configureEach {
    jvmTarget = "17"
}
```

### 3. Create Configuration File

Create `config/detekt/detekt.yml`:

```yaml
build:
  maxIssues: 0
  excludeCorrectable: false
  weights:
    complexity: 2
    LongParameterList: 1
    style: 1
    comments: 1

config:
  validation: true
  warningsAsErrors: false
  checkExhaustiveness: false

processors:
  active: true
  exclude:
    - 'DetektProgressListener'

console-reports:
  active: true
  exclude:
    - 'ProjectStatisticsReport'
    - 'ComplexityReport'
    - 'NotificationReport'
    - 'FindingsReport'
    - 'FileBasedFindingsReport'

output-reports:
  active: true
  exclude:
    - 'TxtOutputReport'
    - 'XmlOutputReport'
    - 'HtmlOutputReport'
    - 'MdOutputReport'

comments:
  active: true
  AbsentOrWrongFileLicense:
    active: false
  CommentOverPrivateFunction:
    active: false
  CommentOverPrivateProperty:
    active: false
  DeprecatedBlockTag:
    active: false
  EndOfSentenceFormat:
    active: false
  OutdatedDocumentation:
    active: false
  UndocumentedPublicClass:
    active: false
  UndocumentedPublicFunction:
    active: false
  UndocumentedPublicProperty:
    active: false

complexity:
  active: true
  ComplexCondition:
    active: true
    threshold: 4
  ComplexInterface:
    active: false
    threshold: 10
    includeStaticDeclarations: false
    includePrivateDeclarations: false
  CyclomaticComplexMethod:
    active: true
    threshold: 15
    ignoreSingleWhenExpression: false
    ignoreSimpleWhenEntries: false
    ignoreNestingFunctions: false
  LabeledExpression:
    active: false
  LargeClass:
    active: true
    threshold: 600
  LongMethod:
    active: true
    threshold: 60
  LongParameterList:
    active: true
    functionThreshold: 6
    constructorThreshold: 7
    ignoreDefaultParameters: false
    ignoreDataClasses: true
    ignoreAnnotatedParameter: []
  MethodOverloading:
    active: false
    threshold: 6
  NamedArguments:
    active: false
    threshold: 3
    ignoreArgumentsMatchingNames: false
  NestedBlockDepth:
    active: true
    threshold: 4
  NestedScopeFunctions:
    active: false
    threshold: 1
    functions:
      - 'kotlin.apply'
      - 'kotlin.run'
      - 'kotlin.with'
      - 'kotlin.let'
      - 'kotlin.also'
  ReplaceSafeCallChainWithRun:
    active: false
  StringLiteralDuplication:
    active: false
    threshold: 3
    ignoreAnnotation: true
    excludeStringsWithLessThan5Characters: true
    ignoreStringsRegex: '$^'
  TooManyFunctions:
    active: true
    thresholdInFiles: 11
    thresholdInClasses: 11
    thresholdInInterfaces: 11
    thresholdInObjects: 11
    thresholdInEnums: 11
    ignoreDeprecated: false
    ignorePrivate: false
    ignoreOverridden: false

coroutines:
  active: true
  GlobalCoroutineUsage:
    active: true
  InjectDispatcher:
    active: true
  RedundantSuspendModifier:
    active: true
  SleepInsteadOfDelay:
    active: true
  SuspendFunWithCoroutineScopeReceiver:
    active: false
  SuspendFunWithFlowReturnType:
    active: true

empty-blocks:
  active: true
  EmptyCatchBlock:
    active: true
    allowedExceptionNameRegex: '_|(ignore|expected).*'
  EmptyClassBlock:
    active: true
  EmptyDefaultConstructor:
    active: true
  EmptyDoWhileBlock:
    active: true
  EmptyElseBlock:
    active: true
  EmptyFinallyBlock:
    active: true
  EmptyForBlock:
    active: true
  EmptyFunctionBlock:
    active: true
    ignoreOverridden: false
  EmptyIfBlock:
    active: true
  EmptyInitBlock:
    active: true
  EmptyKtFile:
    active: true
  EmptySecondaryConstructor:
    active: true
  EmptyTryBlock:
    active: true
  EmptyWhenBlock:
    active: true
  EmptyWhileBlock:
    active: true

exceptions:
  active: true
  ExceptionRaisedInUnexpectedLocation:
    active: true
    methodNames:
      - 'equals'
      - 'finalize'
      - 'hashCode'
      - 'toString'
  InstanceOfCheckForException:
    active: true
  NotImplementedDeclaration:
    active: false
  ObjectExtendsThrowable:
    active: true
  PrintStackTrace:
    active: true
  RethrowCaughtException:
    active: true
  ReturnFromFinally:
    active: true
  SwallowedException:
    active: true
    ignoredExceptionTypes:
      - 'InterruptedException'
      - 'MalformedURLException'
      - 'NumberFormatException'
      - 'ParseException'
    allowedExceptionNameRegex: '_|(ignore|expected).*'
  ThrowingExceptionFromFinally:
    active: true
  ThrowingExceptionInMain:
    active: false
  ThrowingExceptionsWithoutMessageOrCause:
    active: true
    excludedExceptionTypes:
      - 'IllegalArgumentException'
      - 'IllegalStateException'
      - 'IOException'
  ThrowingNewInstanceOfSameException:
    active: true
  TooGenericExceptionCaught:
    active: true
    exceptionNames:
      - 'ArrayIndexOutOfBoundsException'
      - 'Error'
      - 'Exception'
      - 'IllegalMonitorStateException'
      - 'IndexOutOfBoundsException'
      - 'NullPointerException'
      - 'RuntimeException'
      - 'Throwable'
    allowedExceptionNameRegex: '_|(ignore|expected).*'
  TooGenericExceptionThrown:
    active: true
    exceptionNames:
      - 'Error'
      - 'Exception'
      - 'RuntimeException'
      - 'Throwable'

naming:
  active: true
  BooleanPropertyNaming:
    active: false
    allowedPattern: '^(is|has|are)'
  ClassNaming:
    active: true
    classPattern: '[A-Z][a-zA-Z0-9]*'
  ConstructorParameterNaming:
    active: true
    parameterPattern: '[a-z][A-Za-z0-9]*'
    privateParameterPattern: '[a-z][A-Za-z0-9]*'
    excludeClassPattern: '$^'
  EnumNaming:
    active: true
    enumEntryPattern: '[A-Z][_a-zA-Z0-9]*'
  ForbiddenClassName:
    active: false
  FunctionMaxLength:
    active: false
    maximumFunctionNameLength: 30
  FunctionMinLength:
    active: false
    minimumFunctionNameLength: 3
  FunctionNaming:
    active: true
    excludeClassPattern: '$^'
    functionPattern: '[a-z][a-zA-Z0-9]*'
    excludeAnnotatedFunction:
      - 'Composable'
  FunctionParameterNaming:
    active: true
    parameterPattern: '[a-z][A-Za-z0-9]*'
  InvalidPackageDeclaration:
    active: true
  LambdaParameterNaming:
    active: false
    parameterPattern: '[a-z][A-Za-z0-9]*|_'
  MatchingDeclarationName:
    active: true
  MemberNameEqualsClassName:
    active: true
    ignoreOverridden: true
  NoNameShadowing:
    active: true
  NonBooleanPropertyPrefixedWithIs:
    active: false
  ObjectPropertyNaming:
    active: true
    constantPattern: '[A-Za-z][_A-Za-z0-9]*'
    propertyPattern: '[A-Za-z][_A-Za-z0-9]*'
    privatePropertyPattern: '(_)?[A-Za-z][_A-Za-z0-9]*'
  PackageNaming:
    active: true
    packagePattern: '[a-z]+(\.[a-z][A-Za-z0-9]*)*'
  TopLevelPropertyNaming:
    active: true
    constantPattern: '[A-Z][_A-Z0-9]*'
    propertyPattern: '[A-Za-z][_A-Za-z0-9]*'
    privatePropertyPattern: '_?[A-Za-z][_A-Za-z0-9]*'
  VariableMaxLength:
    active: false
    maximumVariableNameLength: 64
  VariableMinLength:
    active: false
    minimumVariableNameLength: 1
  VariableNaming:
    active: true
    variablePattern: '[a-z][A-Za-z0-9]*'
    privateVariablePattern: '(_)?[a-z][A-Za-z0-9]*'
    excludeClassPattern: '$^'

performance:
  active: true
  ArrayPrimitive:
    active: true
  CouldBeSequence:
    active: false
    threshold: 3
  ForEachOnRange:
    active: true
  SpreadOperator:
    active: false
  UnnecessaryPartOfBinaryExpression:
    active: false
  UnnecessaryTemporaryInstantiation:
    active: true

potential-bugs:
  active: true
  AvoidReferentialEquality:
    active: true
  CastToNullableType:
    active: false
  Deprecation:
    active: false
  DontDowncastCollectionTypes:
    active: false
  DoubleMutabilityForCollection:
    active: true
  ElseCaseInsteadOfExhaustiveWhen:
    active: false
  EqualsAlwaysReturnsTrueOrFalse:
    active: true
  EqualsWithHashCodeExist:
    active: true
  ExitOutsideMain:
    active: false
  ExplicitGarbageCollectionCall:
    active: true
  HasPlatformType:
    active: true
  IgnoredReturnValue:
    active: true
  ImplicitDefaultLocale:
    active: true
  ImplicitUnitReturnType:
    active: false
  InvalidRange:
    active: true
  IteratorHasNextCallsNextMethod:
    active: true
  IteratorNotThrowingNoSuchElementException:
    active: true
  LateinitUsage:
    active: false
  MapGetWithNotNullAssertionOperator:
    active: true
  MissingPackageDeclaration:
    active: false
  NullCheckOnMutableProperty:
    active: false
  NullableToStringCall:
    active: false
  UnconditionalJumpStatementInLoop:
    active: false
  UnnecessaryNotNullCheck:
    active: false
  UnnecessaryNotNullOperator:
    active: true
  UnnecessarySafeCall:
    active: true
  UnreachableCatchBlock:
    active: true
  UnreachableCode:
    active: true
  UnsafeCallOnNullableType:
    active: true
  UnsafeCast:
    active: true
  UnusedUnaryOperator:
    active: true
  UselessPostfixExpression:
    active: true
  WrongEqualsTypeParameter:
    active: true

style:
  active: true
  ClassOrdering:
    active: false
  CollapsibleIfStatements:
    active: false
  DataClassContainsFunctions:
    active: false
    conversionFunctionPrefix:
      - 'to'
    allowOperators: false
  DataClassShouldBeImmutable:
    active: false
  DestructuringDeclarationWithTooManyEntries:
    active: true
    maxDestructuringEntries: 3
  EqualsNullCall:
    active: true
  EqualsOnSignatureLine:
    active: false
  ExplicitCollectionElementAccessMethod:
    active: false
  ExplicitItLambdaParameter:
    active: true
  ExpressionBodySyntax:
    active: false
    includeLineWrapping: false
  ForbiddenComment:
    active: false
  ForbiddenImport:
    active: false
  ForbiddenMethodCall:
    active: false
  ForbiddenVoid:
    active: true
  FunctionOnlyReturningConstant:
    active: true
    ignoreOverridableFunction: true
    ignoreActualFunction: true
  LoopWithTooManyJumpStatements:
    active: true
    maxJumpCount: 1
  MagicNumber:
    active: true
    excludes:
      - '**/test/**'
      - '**/androidTest/**'
      - '**/*.Test.kt'
      - '**/*.Spec.kt'
    ignoreNumbers:
      - '-1'
      - '0'
      - '1'
      - '2'
    ignoreHashCodeFunction: true
    ignorePropertyDeclaration: false
    ignoreLocalVariableDeclaration: false
    ignoreConstantDeclaration: true
    ignoreCompanionObjectPropertyDeclaration: true
    ignoreAnnotation: false
    ignoreNamedArgument: true
    ignoreEnums: false
    ignoreRanges: false
    ignoreExtensionFunctions: true
  MandatoryBracesIfStatements:
    active: false
  MandatoryBracesLoops:
    active: false
  MaxChainedCallsOnSameLine:
    active: false
    maxChainedCalls: 5
  MaxLineLength:
    active: true
    maxLineLength: 120
    excludePackageStatements: true
    excludeImportStatements: true
    excludeCommentStatements: false
    excludeRawStrings: true
  MayBeConst:
    active: true
  ModifierOrder:
    active: true
  MultilineLambdaItParameter:
    active: false
  NestedClassesVisibility:
    active: true
  NewLineAtEndOfFile:
    active: true
  NoTabs:
    active: false
  ObjectLiteralToLambda:
    active: true
  OptionalAbstractKeyword:
    active: true
  OptionalUnit:
    active: false
  PreferToOverPairSyntax:
    active: false
  ProtectedMemberInFinalClass:
    active: true
  RedundantExplicitType:
    active: false
  RedundantHigherOrderMapUsage:
    active: true
  RedundantVisibilityModifierRule:
    active: false
  ReturnCount:
    active: true
    max: 2
    excludedFunctions:
      - 'equals'
    excludeLabeled: false
    excludeReturnFromLambda: true
    excludeGuardClauses: false
  SafeCast:
    active: true
  SerialVersionUIDInSerializableClass:
    active: true
  SpacingBetweenPackageAndImports:
    active: false
  ThrowsCount:
    active: true
    max: 2
    excludeGuardClauses: false
  TrailingWhitespace:
    active: false
  UnderscoresInNumericLiterals:
    active: false
  UnnecessaryAbstractClass:
    active: true
  UnnecessaryAnnotationUseSiteTarget:
    active: false
  UnnecessaryApply:
    active: true
  UnnecessaryBackticks:
    active: false
  UnnecessaryBracesAroundTrailingLambda:
    active: false
  UnnecessaryFilter:
    active: true
  UnnecessaryInheritance:
    active: true
  UnnecessaryInnerClass:
    active: false
  UnnecessaryLet:
    active: false
  UnnecessaryParentheses:
    active: false
  UntilInsteadOfRangeTo:
    active: false
  UnusedImports:
    active: false
  UnusedParameter:
    active: true
  UnusedPrivateClass:
    active: true
  UnusedPrivateMember:
    active: true
    allowedNames: '(_|ignored|expected|serialVersionUID)'
  UnusedPrivateProperty:
    active: true
    allowedNames: '_|ignored|expected|serialVersionUID'
  UseAnyOrNoneInsteadOfFind:
    active: true
  UseArrayLiteralsInAnnotations:
    active: true
  UseCheckNotNull:
    active: true
  UseCheckOrError:
    active: true
  UseDataClass:
    active: false
  UseEmptyCounterpart:
    active: false
  UseIfEmptyOrIfBlank:
    active: false
  UseIfInsteadOfWhen:
    active: false
  UseIsNullOrEmpty:
    active: true
  UseOrEmpty:
    active: true
  UseRequire:
    active: true
  UseRequireNotNull:
    active: true
  UseSumOfInsteadOfFlatMapSize:
    active: false
  UselessCallOnNotNull:
    active: true
  UtilityClassWithPublicConstructor:
    active: true
  VarCouldBeVal:
    active: true
  WildcardImport:
    active: true
    excludeImports:
      - 'java.util.*'
```

## Run Detekt

### Command Line

```bash
# Run detekt on all source sets
./gradlew detekt

# Generate baseline
./gradlew detektBaseline

# Auto-fix issues
./gradlew detektFormat
```

### Gradle Task

```kotlin
tasks.register("detektAll") {
    dependsOn(":app:detekt")
    dependsOn(":feature:detekt")
}
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Detekt

on: [push, pull_request]

jobs:
  detekt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Run Detekt
        run: ./gradlew detekt
      
      - name: Upload SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: build/reports/detekt/detekt.sarif
```

## IDE Integration

### IntelliJ IDEA / Android Studio

1. Install Detekt plugin from Marketplace
2. Configure: **Settings → Tools → Detekt**
3. Enable auto-format on save

## Best Practices

- Run detekt in CI/CD pipeline
- Use baseline for legacy code
- Enable formatting plugin
- Customize rules for your project
- Review reports regularly
- Keep configuration updated

## Resources

- [Official Documentation](https://detekt.dev/)
- [Rule Set](https://detekt.dev/docs/rules/complexity)
- [Configuration](https://detekt.dev/docs/introduction/configurations)
- [Custom Rules](https://detekt.dev/docs/introduction/extensions)

