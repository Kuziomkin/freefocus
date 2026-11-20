# Contract: TaskProvider (internal Dart interface)

Internal boundary between screens and task sources (Principle II). Lives in `lib/core/tasks/`. Only `lib/providers/<name>/` implements it. No screen, the timer, or the store imports a provider folder.

## Interface

```dart
abstract interface class TaskProvider {
  /// Stable identifier, e.g. 'todoist'.
  String get id;

  /// Validates the credential and returns the account name.
  /// Throws [ProviderUnauthorized] on rejection, [ProviderUnreachable] on network failure.
  Future<ProviderAccount> connect(String credential);

  /// Open tasks due today or overdue, in provider order.
  /// Throws [ProviderUnauthorized], [ProviderUnreachable].
  Future<List<FocusTask>> fetchTodayTasks();

  /// Projects keyed by ID.
  Future<Map<String, Project>> fetchProjects();

  /// Completes a task. Recurring tasks advance to their next occurrence.
  /// Throws [ProviderUnauthorized], [ProviderUnreachable].
  Future<void> completeTask(String taskId);
}

final class ProviderAccount {
  const ProviderAccount({required this.name, required this.tokenSuffix});
  final String name;
  final String tokenSuffix;
}

sealed class ProviderException implements Exception {
  const ProviderException();
}
final class ProviderUnauthorized extends ProviderException {
  const ProviderUnauthorized();
}
final class ProviderUnreachable extends ProviderException {
  const ProviderUnreachable(this.category);
  final String category; // 'timeout' | 'network' | 'server' | 'rateLimited'
}
```

## Behavioural rules

- `connect` never persists the credential. The caller stores it in the Keychain only after success (FR-017).
- `fetchTodayTasks` pages through every `next_cursor` before returning (contract in [todoist-endpoints.md](todoist-endpoints.md)).
- `completeTask` is called optimistically by the UI. If it throws, the UI rolls back the strike-through and shows an inline error (FR-044).
- Rate-limit retries happen inside the provider; callers see only `ProviderUnreachable('rateLimited')` after the final attempt.
- The provider never throws raw HTTP or JSON exceptions across this boundary.

## Registry

`lib/core/tasks/provider_registry.dart` holds one line per provider:

```dart
final List<TaskProvider Function(CredentialStore)> providerFactories = [
  TodoistProvider.new,
];
```

Adding a provider means one folder and one registry line (FR-007).

## Test double

`test/support/fake_provider.dart` implements `TaskProvider` with scripted results. Every screen test uses it, which proves the UI does not depend on Todoist (Principle II).
