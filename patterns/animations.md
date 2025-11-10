# Animation & Transition Patterns

## Table of Contents
- [Basic Animations](#basic-animations)
- [Animated Visibility](#animated-visibility)
- [AnimatedContent](#animatedcontent)
- [List Animations](#list-animations)
- [Gesture Animations](#gesture-animations)
- [Screen Transitions](#screen-transitions)
- [Shared Element Transitions](#shared-element-transitions)
- [Custom Animations](#custom-animations)

## Basic Animations

### Animate Color

```kotlin
@Composable
fun AnimatedColorExample() {
    var isSelected by remember { mutableStateOf(false) }
    
    val backgroundColor by animateColorAsState(
        targetValue = if (isSelected) {
            MaterialTheme.colorScheme.primary
        } else {
            MaterialTheme.colorScheme.surface
        },
        label = "background color"
    )
    
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable { isSelected = !isSelected },
        colors = CardDefaults.cardColors(containerColor = backgroundColor)
    ) {
        Text("Click me", modifier = Modifier.padding(16.dp))
    }
}
```

### Animate Size

```kotlin
@Composable
fun AnimatedSizeExample() {
    var expanded by remember { mutableStateOf(false) }
    
    val size by animateDpAsState(
        targetValue = if (expanded) 200.dp else 100.dp,
        label = "size"
    )
    
    Box(
        modifier = Modifier
            .size(size)
            .background(MaterialTheme.colorScheme.primary)
            .clickable { expanded = !expanded }
    )
}
```

### Animate Float

```kotlin
@Composable
fun AnimatedAlphaExample() {
    var visible by remember { mutableStateOf(true) }
    
    val alpha by animateFloatAsState(
        targetValue = if (visible) 1f else 0f,
        label = "alpha"
    )
    
    Column {
        Box(
            modifier = Modifier
                .size(100.dp)
                .graphicsLayer { this.alpha = alpha }
                .background(Color.Blue)
        )
        
        Button(onClick = { visible = !visible }) {
            Text("Toggle")
        }
    }
}
```

### Animate Multiple Values

```kotlin
@Composable
fun MultiAnimationExample() {
    var expanded by remember { mutableStateOf(false) }
    
    val size by animateDpAsState(
        targetValue = if (expanded) 200.dp else 100.dp,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow
        ),
        label = "size"
    )
    
    val cornerRadius by animateDpAsState(
        targetValue = if (expanded) 0.dp else 50.dp,
        label = "corner radius"
    )
    
    val backgroundColor by animateColorAsState(
        targetValue = if (expanded) Color.Green else Color.Blue,
        label = "background"
    )
    
    Box(
        modifier = Modifier
            .size(size)
            .clip(RoundedCornerShape(cornerRadius))
            .background(backgroundColor)
            .clickable { expanded = !expanded }
    )
}
```

## Animated Visibility

### Basic AnimatedVisibility

```kotlin
@Composable
fun AnimatedVisibilityExample() {
    var visible by remember { mutableStateOf(true) }
    
    Column {
        Button(onClick = { visible = !visible }) {
            Text("Toggle")
        }
        
        AnimatedVisibility(visible = visible) {
            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp)
            ) {
                Text("This is visible", modifier = Modifier.padding(16.dp))
            }
        }
    }
}
```

### Custom Enter/Exit Animations

```kotlin
@Composable
fun CustomEnterExitExample() {
    var visible by remember { mutableStateOf(false) }
    
    AnimatedVisibility(
        visible = visible,
        enter = slideInVertically(
            initialOffsetY = { -it },
            animationSpec = tween(300)
        ) + fadeIn(animationSpec = tween(300)),
        exit = slideOutVertically(
            targetOffsetY = { it },
            animationSpec = tween(300)
        ) + fadeOut(animationSpec = tween(300))
    ) {
        Card {
            Text("Animated Card", modifier = Modifier.padding(16.dp))
        }
    }
}
```

### Expand/Collapse Animation

```kotlin
@Composable
fun ExpandableCard(item: Item) {
    var expanded by remember { mutableStateOf(false) }
    
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp)
            .animateContentSize()  // Smooth size changes
            .clickable { expanded = !expanded }
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(item.title, style = MaterialTheme.typography.titleMedium)
            
            AnimatedVisibility(visible = expanded) {
                Column {
                    Spacer(modifier = Modifier.height(8.dp))
                    Text(item.description)
                }
            }
        }
    }
}
```

### Staggered List Animation

```kotlin
@Composable
fun StaggeredListAnimation(items: List<Item>) {
    Column {
        items.forEachIndexed { index, item ->
            var visible by remember { mutableStateOf(false) }
            
            LaunchedEffect(Unit) {
                delay(index * 50L)  // Stagger by 50ms
                visible = true
            }
            
            AnimatedVisibility(
                visible = visible,
                enter = fadeIn() + slideInHorizontally()
            ) {
                ItemCard(item)
            }
        }
    }
}
```

## AnimatedContent

### Basic AnimatedContent

```kotlin
@Composable
fun AnimatedContentExample() {
    var count by remember { mutableStateOf(0) }
    
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        AnimatedContent(
            targetState = count,
            label = "count animation"
        ) { targetCount ->
            Text(
                text = "$targetCount",
                style = MaterialTheme.typography.displayLarge
            )
        }
        
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

### Slide Transitions

```kotlin
@Composable
fun SlideAnimatedContent(currentScreen: Screen) {
    AnimatedContent(
        targetState = currentScreen,
        transitionSpec = {
            if (targetState > initialState) {
                // Forward navigation
                slideInHorizontally { width -> width } + fadeIn() togetherWith
                        slideOutHorizontally { width -> -width } + fadeOut()
            } else {
                // Back navigation
                slideInHorizontally { width -> -width } + fadeIn() togetherWith
                        slideOutHorizontally { width -> width } + fadeOut()
            }.using(SizeTransform(clip = false))
        },
        label = "screen transition"
    ) { screen ->
        ScreenContent(screen)
    }
}
```

### Scale and Fade

```kotlin
@Composable
fun ScaleFadeContent(isExpanded: Boolean) {
    AnimatedContent(
        targetState = isExpanded,
        transitionSpec = {
            if (targetState) {
                scaleIn() + fadeIn() togetherWith scaleOut() + fadeOut()
            } else {
                scaleOut() + fadeOut() togetherWith scaleIn() + fadeIn()
            }
        },
        label = "scale fade"
    ) { expanded ->
        if (expanded) {
            DetailedView()
        } else {
            CompactView()
        }
    }
}
```

## List Animations

### Animated LazyColumn

```kotlin
@Composable
fun AnimatedList(items: List<Item>) {
    LazyColumn {
        items(
            items = items,
            key = { it.id }
        ) { item ->
            AnimatedListItem(item)
        }
    }
}

@Composable
fun AnimatedListItem(item: Item) {
    var visible by remember { mutableStateOf(false) }
    
    LaunchedEffect(Unit) {
        visible = true
    }
    
    AnimatedVisibility(
        visible = visible,
        enter = slideInVertically() + fadeIn()
    ) {
        Card(
            modifier = Modifier
                .fillMaxWidth()
                .padding(8.dp)
                .animateItemPlacement()  // Smooth reordering
        ) {
            Text(item.name, modifier = Modifier.padding(16.dp))
        }
    }
}
```

### Item Removal Animation

```kotlin
@Composable
fun RemovableListItem(
    item: Item,
    onRemove: () -> Unit
) {
    var visible by remember { mutableStateOf(true) }
    
    AnimatedVisibility(
        visible = visible,
        exit = shrinkVertically() + fadeOut()
    ) {
        SwipeToDismissBox(
            state = rememberSwipeToDismissBoxState(
                confirmValueChange = {
                    visible = false
                    true
                }
            ),
            backgroundContent = { /* Delete background */ }
        ) {
            Card {
                Text(item.name, modifier = Modifier.padding(16.dp))
            }
        }
    }
    
    LaunchedEffect(visible) {
        if (!visible) {
            delay(300)  // Wait for animation
            onRemove()
        }
    }
}
```

## Gesture Animations

### Swipe Animation

```kotlin
@Composable
fun SwipeableCard() {
    var offsetX by remember { mutableStateOf(0f) }
    
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .offset { IntOffset(offsetX.roundToInt(), 0) }
            .pointerInput(Unit) {
                detectHorizontalDragGestures(
                    onDragEnd = {
                        offsetX = 0f  // Spring back
                    },
                    onHorizontalDrag = { _, dragAmount ->
                        offsetX += dragAmount
                    }
                )
            }
    ) {
        Text("Swipe me", modifier = Modifier.padding(16.dp))
    }
}
```

### Pull Down to Refresh Animation

```kotlin
@Composable
fun CustomPullToRefresh(
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
    content: @Composable () -> Unit
) {
    var offsetY by remember { mutableStateOf(0f) }
    val maxDrag = 200f
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectVerticalDragGestures(
                    onDragEnd = {
                        if (offsetY > maxDrag) {
                            onRefresh()
                        }
                        offsetY = 0f
                    },
                    onVerticalDrag = { _, dragAmount ->
                        offsetY = (offsetY + dragAmount).coerceIn(0f, maxDrag * 1.5f)
                    }
                )
            }
    ) {
        Box(modifier = Modifier.offset { IntOffset(0, offsetY.toInt()) }) {
            content()
        }
        
        if (offsetY > 0) {
            val progress = (offsetY / maxDrag).coerceIn(0f, 1f)
            CircularProgressIndicator(
                progress = { progress },
                modifier = Modifier
                    .align(Alignment.TopCenter)
                    .padding(top = 16.dp)
            )
        }
    }
}
```

### Draggable Item

```kotlin
@Composable
fun DraggableBox() {
    var offsetX by remember { mutableStateOf(0f) }
    var offsetY by remember { mutableStateOf(0f) }
    
    Box(
        modifier = Modifier
            .offset { IntOffset(offsetX.roundToInt(), offsetY.roundToInt()) }
            .size(100.dp)
            .background(Color.Blue)
            .pointerInput(Unit) {
                detectDragGestures { change, dragAmount ->
                    change.consume()
                    offsetX += dragAmount.x
                    offsetY += dragAmount.y
                }
            }
    )
}
```

## Screen Transitions

### Fade Transition

```kotlin
@Composable
fun FadeTransition(screen: Screen) {
    val transition = updateTransition(targetState = screen, label = "screen")
    
    transition.AnimatedContent(
        transitionSpec = {
            fadeIn(animationSpec = tween(300)) togetherWith
                    fadeOut(animationSpec = tween(300))
        }
    ) { targetScreen ->
        ScreenContent(targetScreen)
    }
}
```

### Slide Transition

```kotlin
@Composable
fun SlideTransition(screen: Screen) {
    AnimatedContent(
        targetState = screen,
        transitionSpec = {
            slideIntoContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            ) togetherWith slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            )
        },
        label = "slide transition"
    ) { targetScreen ->
        ScreenContent(targetScreen)
    }
}
```

### Scale Transition

```kotlin
@Composable
fun ScaleTransition() {
    var visible by remember { mutableStateOf(false) }
    
    Column {
        Button(onClick = { visible = !visible }) {
            Text("Toggle")
        }
        
        AnimatedVisibility(
            visible = visible,
            enter = scaleIn(
                initialScale = 0.3f,
                animationSpec = spring(
                    dampingRatio = Spring.DampingRatioMediumBouncy
                )
            ),
            exit = scaleOut(targetScale = 0.3f)
        ) {
            Card {
                Text("Animated Card", modifier = Modifier.padding(32.dp))
            }
        }
    }
}
```

## Shared Element Transitions

### Basic Shared Element

```kotlin
@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun SharedElementExample() {
    var showDetails by remember { mutableStateOf(false) }
    
    SharedTransitionLayout {
        AnimatedContent(
            targetState = showDetails,
            label = "shared element"
        ) { isShowingDetails ->
            if (!isShowingDetails) {
                ListScreen(
                    onItemClick = { showDetails = true }
                )
            } else {
                DetailScreen(
                    onBack = { showDetails = false }
                )
            }
        }
    }
}

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun SharedTransitionScope.ListItem(
    item: Item,
    animatedVisibilityScope: AnimatedVisibilityScope,
    onClick: () -> Unit
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable(onClick = onClick)
    ) {
        Row {
            AsyncImage(
                model = item.imageUrl,
                contentDescription = null,
                modifier = Modifier
                    .size(80.dp)
                    .sharedElement(
                        state = rememberSharedContentState(key = "image-${item.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
            )
            
            Text(
                text = item.name,
                modifier = Modifier
                    .padding(16.dp)
                    .sharedElement(
                        state = rememberSharedContentState(key = "text-${item.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
            )
        }
    }
}
```

## Custom Animations

### Pulsing Animation

```kotlin
@Composable
fun PulsingDot() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")
    
    val scale by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 1.3f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )
    
    Box(
        modifier = Modifier
            .size(20.dp)
            .scale(scale)
            .background(Color.Red, CircleShape)
    )
}
```

### Rotating Loading Indicator

```kotlin
@Composable
fun RotatingLoadingIndicator() {
    val infiniteTransition = rememberInfiniteTransition(label = "rotate")
    
    val rotation by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 360f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "rotation"
    )
    
    Icon(
        imageVector = Icons.Default.Refresh,
        contentDescription = "Loading",
        modifier = Modifier
            .size(48.dp)
            .rotate(rotation)
    )
}
```

### Shimmer Effect

```kotlin
@Composable
fun ShimmerEffect() {
    val infiniteTransition = rememberInfiniteTransition(label = "shimmer")
    
    val offset by infiniteTransition.animateFloat(
        initialValue = -1000f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(1500, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "offset"
    )
    
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .background(
                Brush.horizontalGradient(
                    colors = listOf(
                        Color.LightGray.copy(alpha = 0.3f),
                        Color.LightGray.copy(alpha = 0.5f),
                        Color.LightGray.copy(alpha = 0.3f)
                    ),
                    startX = offset,
                    endX = offset + 500f
                )
            )
    )
}
```

### Bouncing Animation

```kotlin
@Composable
fun BouncingBall() {
    val infiniteTransition = rememberInfiniteTransition(label = "bounce")
    
    val offsetY by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 100f,
        animationSpec = infiniteRepeatable(
            animation = tween(500, easing = FastOutSlowInEasing),
            repeatMode = RepeatMode.Reverse
        ),
        label = "bounce"
    )
    
    Box(
        modifier = Modifier
            .size(50.dp)
            .offset(y = offsetY.dp)
            .background(Color.Blue, CircleShape)
    )
}
```

## Animation Specs

### Spring Animation

```kotlin
@Composable
fun SpringAnimationExample() {
    var expanded by remember { mutableStateOf(false) }
    
    val size by animateDpAsState(
        targetValue = if (expanded) 200.dp else 100.dp,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioHighBouncy,
            stiffness = Spring.StiffnessMedium
        ),
        label = "spring size"
    )
    
    Box(
        modifier = Modifier
            .size(size)
            .background(Color.Blue)
            .clickable { expanded = !expanded }
    )
}
```

### Tween Animation

```kotlin
val size by animateDpAsState(
    targetValue = if (expanded) 200.dp else 100.dp,
    animationSpec = tween(
        durationMillis = 500,
        delayMillis = 100,
        easing = FastOutSlowInEasing
    ),
    label = "tween size"
)
```

### Keyframes Animation

```kotlin
val size by animateDpAsState(
    targetValue = if (expanded) 200.dp else 100.dp,
    animationSpec = keyframes {
        durationMillis = 1000
        100.dp at 0 using LinearEasing
        150.dp at 500 using FastOutSlowInEasing
        200.dp at 1000
    },
    label = "keyframe size"
)
```

### Repeatable Animation

```kotlin
val infiniteTransition = rememberInfiniteTransition(label = "infinite")

val alpha by infiniteTransition.animateFloat(
    initialValue = 0.3f,
    targetValue = 1f,
    animationSpec = infiniteRepeatable(
        animation = tween(1000),
        repeatMode = RepeatMode.Reverse
    ),
    label = "alpha"
)
```

## Progress Animations

### Progress Bar Animation

```kotlin
@Composable
fun AnimatedProgressBar(progress: Float) {
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(durationMillis = 1000),
        label = "progress"
    )
    
    LinearProgressIndicator(
        progress = { animatedProgress },
        modifier = Modifier.fillMaxWidth()
    )
}
```

### Circular Progress Animation

```kotlin
@Composable
fun AnimatedCircularProgress(progress: Float) {
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(1000),
        label = "circular progress"
    )
    
    Box(contentAlignment = Alignment.Center) {
        CircularProgressIndicator(
            progress = { animatedProgress },
            modifier = Modifier.size(100.dp),
            strokeWidth = 8.dp
        )
        
        Text(
            text = "${(animatedProgress * 100).toInt()}%",
            style = MaterialTheme.typography.titleLarge
        )
    }
}
```

## Success Animations

### Checkmark Animation

```kotlin
@Composable
fun AnimatedCheckmark() {
    var visible by remember { mutableStateOf(false) }
    
    LaunchedEffect(Unit) {
        delay(300)
        visible = true
    }
    
    Box(
        modifier = Modifier.size(100.dp),
        contentAlignment = Alignment.Center
    ) {
        AnimatedVisibility(
            visible = visible,
            enter = scaleIn(
                animationSpec = spring(
                    dampingRatio = Spring.DampingRatioMediumBouncy
                )
            ) + fadeIn()
        ) {
            Icon(
                imageVector = Icons.Default.CheckCircle,
                contentDescription = "Success",
                modifier = Modifier.size(80.dp),
                tint = Color.Green
            )
        }
    }
}
```

## Performance Tips

### Optimize Animations

```kotlin
@Composable
fun OptimizedAnimation() {
    var expanded by remember { mutableStateOf(false) }
    
    // Use graphicsLayer for better performance
    Box(
        modifier = Modifier
            .size(100.dp)
            .graphicsLayer {
                scaleX = if (expanded) 1.5f else 1f
                scaleY = if (expanded) 1.5f else 1f
                alpha = if (expanded) 1f else 0.5f
            }
            .background(Color.Blue)
            .clickable { expanded = !expanded }
    )
}
```

### Defer Heavy Animations

```kotlin
@Composable
fun DeferredAnimation() {
    var shouldAnimate by remember { mutableStateOf(false) }
    
    LaunchedEffect(Unit) {
        delay(100)  // Defer animation until screen is loaded
        shouldAnimate = true
    }
    
    AnimatedVisibility(visible = shouldAnimate) {
        HeavyAnimatedContent()
    }
}
```

## Best Practices

- Use animateContentSize() for smooth size changes
- Use animateItemPlacement() for list reordering
- Provide label parameter for debugging
- Use graphicsLayer for transform animations
- Use infinite transitions sparingly
- Clean up animations in DisposableEffect
- Test on low-end devices
- Use appropriate animation specs
- Consider accessibility preferences

## Common Mistakes

### Animating Inside Composition

```kotlin
// Bad
@Composable
fun BadAnimation() {
    animate(0f, 1f)  // Don't animate during composition
}

// Good
@Composable
fun GoodAnimation() {
    var target by remember { mutableStateOf(0f) }
    val animated by animateFloatAsState(target, label = "value")
}
```

### Creating New Transition States

```kotlin
// Bad - Creates new state on every recomposition
@Composable
fun BadTransition() {
    val transition = updateTransition(State(), label = "transition")
}

// Good - Remember the state
@Composable
fun GoodTransition() {
    val state = remember { mutableStateOf(State()) }
    val transition = updateTransition(state.value, label = "transition")
}
```

## Resources

- [Compose Animation](https://developer.android.com/jetpack/compose/animation)
- [Animation Specs](https://developer.android.com/jetpack/compose/animation/customize)
- [Shared Element Transitions](https://developer.android.com/jetpack/compose/animation/shared-elements)
- [Animation Cookbook](https://developer.android.com/jetpack/compose/animation-cookbook)

