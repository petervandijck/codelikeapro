# Pinkary Design Patterns and Best Practices Analysis

## 1. Interface Implementation Pattern
This pattern defines contracts (interfaces) that classes must implement, ensuring consistent behavior across different implementations.

Example from `App/Contracts/Models/Viewable.php`:
```php
interface Viewable
{
    // Defines a contract that any model supporting view counts must implement
    public static function incrementViews(array $ids): void;
}
```

Used in `App/Models/User.php` and `App/Models/Question.php`:
```php
// Both classes implement the same interface, ensuring consistent view counting behavior
class User extends Authenticatable implements FilamentUser, MustVerifyEmail, Viewable
{
    public static function incrementViews(array $ids): void
    {
        self::withoutTimestamps(function () use ($ids): void {
            self::query()
                ->whereIn('id', $ids)
                ->increment('views');
        });
    }
}
```

[Laravel Contracts Documentation](https://laravel.com/docs/11.x/contracts)

## 2. Job Pattern
Jobs are used for handling background tasks and heavy operations outside the main request cycle.

Example from `App/Jobs/UpdateUserAvatar.php`:
```php
final class UpdateUserAvatar implements ShouldQueue
{
    use Queueable;

    // Constructor injects dependencies
    public function __construct(
        private readonly User $user,
        private readonly ?string $file = null,
        private readonly ?string $service = null
    ) {}

    // Main job logic
    public function handle(): void
    {
        // Avatar processing logic
    }

    // Error handling
    public function failed(?Throwable $exception): void
    {
        // Cleanup on failure
    }
}
```

Used in `App/Http/Controllers/UserAvatarController.php`:
```php
// Dispatches the job synchronously
UpdateUserAvatar::dispatchSync($user, $file->getRealPath());
```

[Laravel Jobs Documentation](https://laravel.com/docs/11.x/queues)

## 3. Observer Pattern
Observers handle model events like created, updated, and deleted, keeping logic organized and separated.

Example from `App/Observers/QuestionObserver.php`:
```php
final readonly class QuestionObserver
{
    // Handles what happens after a question is created
    public function created(Question $question): void
    {
        if ($question->isSharedUpdate()) {
            // Handle shared update logic
        } else {
            // Handle normal question creation
            $question->loadMissing('to');
            $question->to->notify(new QuestionCreated($question));
        }
    }
}
```

Used via attribute in `App/Models/Question.php`:
```php
#[ObservedBy(QuestionObserver::class)]
final class Question extends Model implements Viewable
```

[Laravel Model Events Documentation](https://laravel.com/docs/11.x/eloquent#events)

## 4. Policy Pattern
Policies centralize authorization logic for models.

Example from `App/Policies/QuestionPolicy.php`:
```php
final readonly class QuestionPolicy
{
    // Determines if a user can view a question
    public function view(?User $user, Question $question): bool
    {
        if ($question->is_ignored || $question->is_reported) {
            return false;
        }

        return $question->answer || $question->to_id === $user?->id;
    }
}
```

Used in controllers via the Gate facade:
```php
// In QuestionController.php
Gate::authorize('view', $question);
```

[Laravel Authorization Documentation](https://laravel.com/docs/11.x/authorization)

## 5. Command Pattern
Artisan commands encapsulate CLI operations in dedicated classes.

Example from `App/Console/Commands/DeleteNonEmailVerifiedUsersCommand.php`:
```php
final class DeleteNonEmailVerifiedUsersCommand extends Command
{
    // Command signature defines how to call it
    protected $signature = 'delete:non-email-verified-users';

    // Main command logic
    public function handle(): void
    {
        User::where('email_verified_at', null)
            ->where('updated_at', '<', now()->subDay())
            ->whereDoesntHave('links')
            // ... additional conditions
            ->each->purge();
    }
}
```

[Laravel Artisan Commands Documentation](https://laravel.com/docs/11.x/artisan)

## 6. Service Pattern
Services encapsulate complex business logic in dedicated classes.

Example from `App/Services/ParsableContent.php` interface:
```php
interface ParsableContentProvider
{
    // Defines a contract for parsing content
    public function parse(string $content): string;
}
```

Used in `App/Models/Question.php`:
```php
public function getContentAttribute(?string $value): ?string
{
    $content = new ParsableContent();
    return $value !== null ? $content->parse($value) : null;
}
```

## 7. Enum Pattern
Enums provide type-safe value constraints.

Example from `App/Enums/UserMailPreference.php`:
```php
enum UserMailPreference: string
{
    case Daily = 'daily';
    case Weekly = 'weekly';
    case Never = 'never';

    // Helper method to convert enum to array
    public static function toArray(): array
    {
        return [
            self::Daily->value => 'Daily',
            self::Weekly->value => 'Weekly',
            self::Never->value => 'Never',
        ];
    }
}
```

Used in `App/Models/User.php`:
```php
protected function casts(): array
{
    return [
        'mail_preference_time' => UserMailPreference::class,
        // ... other casts
    ];
}
```

[Laravel Enums Documentation](https://laravel.com/docs/11.x/eloquent-mutators#enum-casting)

## 8. Form Request Pattern
Form requests handle validation logic separately from controllers.

Example from `App/Http/Requests/UserAvatarUpdateRequest.php`:
```php
final class UserAvatarUpdateRequest extends FormRequest
{
    // Defines validation rules
    public function rules(): array
    {
        return [
            'avatar' => ['required', 'image', 'mimes:jpg,jpeg,png', 'max:2048'],
        ];
    }

    // Custom error messages
    public function messages(): array
    {
        return [
            'avatar.max' => 'The avatar may not be greater than 2MB.',
        ];
    }
}
```

Used in `App/Http/Controllers/UserAvatarController.php`:
```php
public function update(UserAvatarUpdateRequest $request): RedirectResponse
```

[Laravel Form Request Validation Documentation](https://laravel.com/docs/11.x/validation#form-request-validation)

## Best Practices Observed

1. **Type Safety**
    - Extensive use of type hints
    - Strict types declaration
    - Return type declarations

2. **Immutability**
    - Use of `readonly` classes
    - Final classes to prevent inheritance where not needed

3. **Single Responsibility**
    - Each class has a clear, focused purpose
    - Logic is separated into appropriate layers (Controllers, Models, Services, etc.)

4. **Dependency Injection**
    - Constructor injection for required dependencies
    - Type-hinted dependencies for better IDE support

5. **Documentation**
    - Clear PHPDoc blocks
    - Property and return type documentation
    - Relationship documentation in models

These patterns demonstrate Laravel's emphasis on clean, maintainable, and well-structured code while providing robust solutions to common web development challenges.
