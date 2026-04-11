# Sub-Agent: Builder UI

## Role
Generate the complete Flutter project structure, all screens, shared widgets, and state management wiring. Called by `builder.md` as Phase 1.

## Inputs
Read these fields from `system_plan.json`:
- `frontend.pages` → generate one screen file per page
- `backend.features` → generate one feature folder per feature
- `state_management` → riverpod (default) or bloc
- `deployment_target` → affects pubspec platform config

---

## Project Scaffold (always generate exactly this)

```
lib/
  main.dart                       # Firebase init + ProviderScope + runApp
  app.dart                        # MaterialApp.router + GoRouter config
  core/
    constants/
      app_colors.dart
      app_strings.dart
      app_routes.dart             # route name constants (one per page in frontend.pages)
    utils/
      validators.dart
      formatters.dart
    errors/
      app_exceptions.dart         # typed failures — never use raw strings
  features/
    [feature.name]/               # one folder per entry in backend.features
      data/
        [feature]_model.dart      # freezed + json_serializable
        [feature]_repository.dart # all Firebase calls live here
      logic/
        [feature]_provider.dart   # Riverpod notifier or BLoC cubit
        [feature]_state.dart      # freezed state class
      ui/
        [feature]_screen.dart     # matches page in frontend.pages
        widgets/                  # screen-specific widgets only
  shared/
    widgets/
      loading_indicator.dart      # always generate
      error_view.dart             # always generate
      empty_state_view.dart       # always generate
    services/
      firebase_service.dart       # single Firebase init + service accessors
```

### Rules
- One feature folder per entry in `system_plan.json`.backend.features — no exceptions
- One screen file per entry in `system_plan.json`.frontend.pages
- `app_exceptions.dart` always generated — typed error contract between layers
- `firebase_service.dart` initializes ALL services from `backend.firebase_services`
- Zero business logic in any `ui/` file

---

## pubspec.yaml (always generate)

```yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^3.1.0
  firebase_auth: ^5.1.0
  cloud_firestore: ^5.1.0
  firebase_storage: ^12.1.0
  cloud_functions: ^5.0.0
  flutter_riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5
  go_router: ^14.2.0
  freezed_annotation: ^2.4.1
  json_annotation: ^4.9.0
  fpdart: ^1.1.0              # Either type

dev_dependencies:
  build_runner: ^2.4.11
  freezed: ^2.5.2
  riverpod_generator: ^2.4.0
  json_serializable: ^6.8.0
  flutter_lints: ^4.0.0
```

Only include Firebase packages that appear in `backend.firebase_services`.

---

## main.dart (always generate this exact pattern)

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized(); // ← FL_001 prevention
  await FirebaseService.initialize();
  runApp(const ProviderScope(child: App()));
}
```

---

## app_routes.dart (always generate)

```dart
// core/constants/app_routes.dart
abstract class AppRoutes {
  // generated from system_plan.json frontend.pages — one const per page
  static const String [pageName] = '/[page.route]';
}
```

---

## app.dart — GoRouter as Riverpod provider

GoRouter **must** be a Riverpod provider so `ref` is available for auth redirect:

```dart
// app.dart
@riverpod
GoRouter router(RouterRef ref) {
  final authState = ref.watch(authStateProvider);

  return GoRouter(
    initialLocation: AppRoutes.login, // first page where requires_auth == false
    refreshListenable: _RouterNotifier(ref),
    redirect: (context, state) {
      final loggedIn = authState.valueOrNull != null;
      final goingToAuth = state.matchedLocation == AppRoutes.login;
      if (!loggedIn && !goingToAuth) return AppRoutes.login;
      if (loggedIn && goingToAuth) return AppRoutes.home;
      return null;
    },
    routes: [
      // generated from frontend.pages — one GoRoute per page
      GoRoute(
        name: '[page.name]',
        path: AppRoutes.[pageName],
        builder: (context, state) => const [PageName]Screen(),
      ),
    ],
  );
}

// Bridges Riverpod auth stream → GoRouter refresh
class _RouterNotifier extends ChangeNotifier {
  _RouterNotifier(Ref ref) {
    ref.listen(authStateProvider, (_, __) => notifyListeners());
  }
}

class App extends ConsumerWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);
    return MaterialApp.router(
      routerConfig: router,
      theme: ThemeData(useMaterial3: true),
    );
  }
}
```

---

## State Management

### Default: Riverpod (code-gen style)

```dart
// [feature]_state.dart
@freezed
class [Feature]State with _$[Feature]State {
  const factory [Feature]State.initial() = _Initial;
  const factory [Feature]State.loading() = _Loading;
  const factory [Feature]State.data([Feature] data) = _Data;
  const factory [Feature]State.error(AppException error) = _Error;
}

// [feature]_provider.dart
@riverpod
class [Feature]Notifier extends _$[Feature]Notifier {
  @override
  [Feature]State build() => const [Feature]State.initial();

  Future<void> load() async {
    state = const [Feature]State.loading();
    final result = await ref.read([feature]RepositoryProvider).fetch();
    state = result.fold(
      (failure) => [Feature]State.error(failure),
      (data)    => [Feature]State.data(data),
    );
  }
}
```

### If state_management == "bloc"

```dart
// [feature]_state.dart
@freezed
class [Feature]State with _$[Feature]State {
  const factory [Feature]State.initial() = _Initial;
  const factory [Feature]State.loading() = _Loading;
  const factory [Feature]State.data([Feature] data) = _Data;
  const factory [Feature]State.error(AppException error) = _Error;
}

// [feature]_cubit.dart
class [Feature]Cubit extends Cubit<[Feature]State> {
  final [Feature]Repository _repository;

  [Feature]Cubit(this._repository) : super(const [Feature]State.initial());

  Future<void> load() async {
    emit(const [Feature]State.loading());
    final result = await _repository.fetch();
    result.fold(
      (failure) => emit([Feature]State.error(failure)),
      (data)    => emit([Feature]State.data(data)),
    );
  }
}

// main.dart addition for BLoC
// Replace ProviderScope with MultiBlocProvider at root
```

**Never mix Riverpod and BLoC in the same project.**

---

## app_exceptions.dart (always generate)

```dart
sealed class AppException {
  const AppException();
}
class NetworkException extends AppException {
  final String message;
  const NetworkException(this.message);
}
class AuthException extends AppException {
  final String code;
  const AuthException(this.code);
}
class DatabaseException extends AppException {
  final String message;
  const DatabaseException(this.message);
}
// Only include if 'ml' appears in backend.features:
class MLException extends AppException {
  final String message;
  const MLException(this.message);
}
```

---

## shared/widgets (always generate these three)

```dart
// shared/widgets/loading_indicator.dart
class LoadingIndicator extends StatelessWidget {
  const LoadingIndicator({super.key});
  @override
  Widget build(BuildContext context) =>
      const Center(child: CircularProgressIndicator());
}

// shared/widgets/error_view.dart
class ErrorView extends StatelessWidget {
  final AppException error;
  final VoidCallback? onRetry;
  const ErrorView({super.key, required this.error, this.onRetry});
  @override
  Widget build(BuildContext context) => Center(
    child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Text(error.toString()),
        if (onRetry != null)
          TextButton(onPressed: onRetry, child: const Text('Retry')),
      ],
    ),
  );
}

// shared/widgets/empty_state_view.dart
class EmptyStateView extends StatelessWidget {
  final String message;
  const EmptyStateView({super.key, required this.message});
  @override
  Widget build(BuildContext context) =>
      Center(child: Text(message));
}
```

---

## Error Codes (report to error_handler.md)
| Code | Trigger |
|---|---|
| `FL_001` | `WidgetsFlutterBinding.ensureInitialized()` missing in `main.dart` |
| `FL_002` | Null check on null — fix model with nullable types |
| `FL_003` | `StateError: Stream closed` — add `ref.onDispose` cleanup |
| `FL_004` | Build runner conflict — run with `--delete-conflicting-outputs` |
| `FL_005` | Provider not found — check `ProviderScope` wraps `App` in `main.dart` |
| `FL_006` | GoRouter redirect loop — check both auth guard conditions in router provider |