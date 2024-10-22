# Lessons from the pinkary.com code

## Design Patterns

### 1. Observer Pattern
The Observer pattern is used to handle events and notifications when models change. Laravel provides a built-in way to implement observers.

Example from `App/Observers/QuestionObserver.php`:
```php
// This observer watches for changes on the Question model
public function updated(Question $question): void
{
    // When a question is updated, this code runs
    $question->loadMissing('from', 'to');

    if ($question->is_ignored || $question->is_reported) {
        $this->deleted($question);
        return;
    }
    
    // Remove notifications when question is answered
    if ($question->answer !== null) {
        $question->to->notifications()->whereJsonContains('data->question_id', $question->id)->delete();
    }
}
```

Usage in `App/Models/Question.php`:
```php
#[ObservedBy(QuestionObserver::class)]
final class Question extends Model implements Viewable
```

[Laravel Events Documentation](https://laravel.com/docs/11.x/events)

### 2. Interface Implementation
Interfaces are used to define contracts that classes must fulfill. This ensures consistent behavior across different implementations.

Example from `App/Contracts/Models/Viewable.php`:
```php
// This interface defines a contract for models that can be viewed
interface Viewable
{
    // Any class implementing this interface must have this method
    public static function incrementViews(array $ids): void;
}
```

Implementation in `App/Models/Question.php`:
```php
final class Question extends Model implements Viewable
{
    public static function incrementViews(array $ids): void
    {
        self::withoutTimestamps(function () use ($ids): void {
            self::query()
                ->whereIn('id', $ids)
                ->whereNotNull('answer')
                ->increment('views');
        });
    }
}
```

### 3. Job Pattern
Jobs are used for handling background tasks and queued operations. They help improve application performance by processing heavy tasks asynchronously.

Example from `App/Jobs/UpdateUserAvatar.php`:
```php
// This job handles avatar updates in the background
final class UpdateUserAvatar implements ShouldQueue
{
    use Queueable;

    public function __construct(
        private readonly User $user,
        private readonly ?string $file = null,
        private readonly ?string $service = null
    ) {}

    public function handle(): void
    {
        // Avatar processing logic
    }
}
```

Usage in `App/Http/Controllers/UserAvatarController.php`:
```php
UpdateUserAvatar::dispatchSync($user, $file->getRealPath());
```

[Laravel Queues Documentation](https://laravel.com/docs/11.x/queues)

### 4. Policy Pattern
Policies are used to organize authorization logic around a particular model or resource.

Example from `App/Policies/QuestionPolicy.php`:
```php
// This policy defines authorization rules for Question-related actions
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

Usage through Laravel's Gate facade:
```php
Gate::authorize('view', $question);
```

[Laravel Authorization Documentation](https://laravel.com/docs/11.x/authorization)

### 5. Repository Pattern with Eloquent
While not explicitly using repository classes, the code uses Eloquent models as repositories, encapsulating data access logic.

Example from `App/Models/User.php`:
```php
// Model acts as a repository, encapsulating data access logic
final class User extends Authenticatable implements FilamentUser, MustVerifyEmail, Viewable
{
    public function following(): BelongsToMany
    {
        return $this->belongsToMany(self::class, 'followers', 'follower_id', 'user_id');
    }

    public function followers(): BelongsToMany
    {
        return $this->belongsToMany(self::class, 'followers', 'user_id', 'follower_id');
    }
}
```

### 6. Command Pattern
Commands are used to encapsulate specific tasks that can be executed from the command line.

Example from `App/Console/Commands/DeleteNonEmailVerifiedUsersCommand.php`:
```php
// This command deletes unverified users after 24 hours
final class DeleteNonEmailVerifiedUsersCommand extends Command
{
    protected $signature = 'delete:non-email-verified-users';
    
    public function handle(): void
    {
        User::where('email_verified_at', null)
            ->where('updated_at', '<', now()->subDay())
            ->whereDoesntHave('links')
            ->get()
            ->each
            ->purge();
    }
}
```

[Laravel Artisan Console Documentation](https://laravel.com/docs/11.x/artisan)

### 7. Notification Pattern
Notifications are used to send various types of notifications across multiple channels.

Example from `App/Notifications/QuestionAnswered.php`:
```php
// This notification is sent when a question is answered
final class QuestionAnswered extends Notification
{
    use Queueable;

    public function __construct(private Question $question) {}

    public function via(object $notifiable): array
    {
        return ['database'];
    }
}
```

Usage in `App/Observers/QuestionObserver.php`:
```php
$question->from->notify(new QuestionAnswered($question));
```

[Laravel Notifications Documentation](https://laravel.com/docs/11.x/notifications)

### 8. Form Request Pattern
Form requests are used to encapsulate validation logic for incoming HTTP requests.

Example from `App/Http/Requests/UserUpdateRequest.php`:
```php
// This class handles validation for user updates
final class UserUpdateRequest extends FormRequest
{
    public function rules(): array
    {
        $user = type($this->user())->as(User::class);

        return [
            'name' => ['required', 'string', 'max:255', new NoBlankCharacters],
            'username' => [
                'required', 'string', 'min:4', 'max:50', 
                Rule::unique(User::class)->ignore($user->id),
                new Username($user),
            ],
        ];
    }
}
```

[Laravel Form Request Validation Documentation](https://laravel.com/docs/11.x/validation#form-request-validation)

### 9. Service Pattern
Services are used to encapsulate complex business logic.

Example from `App/Services/ParsableContent.php` (referenced but not shown in provided code):
```php
// Called from Question model to parse content
public function getContentAttribute(?string $value): ?string
{
    $content = new ParsableContent();
    return $value !== null && $value !== '' && $value !== '0' ? 
        $content->parse($value) : null;
}
```

### 10. Middleware Pattern
Middleware provides a way to filter HTTP requests entering the application.

Example from `App/Http/Middleware/EnsureVerifiedEmailsForSignInUsers.php`:
```php
// This middleware ensures users have verified their email
final readonly class EnsureVerifiedEmailsForSignInUsers
{
    public function handle(Request $request, Closure $next): Response
    {
        if (! auth()->check()) {
            return $next($request);
        }

        $user = type($request->user())->as(User::class);

        if ($user->hasVerifiedEmail()) {
            return $next($request);
        }

        return to_route('verification.notice');
    }
}
```

[Laravel Middleware Documentation](https://laravel.com/docs/11.x/middleware)

## Best Practices

### 1. Type Declaration and Strict Typing
```php
declare(strict_types=1);

final readonly class QuestionPolicy
{
    public function view(?User $user, Question $question): bool
```

### 2. Using Value Objects for Complex Types
```php
public function getAvatarUrlAttribute(): string
{
    return $this->avatar ? 
        Storage::disk('public')->url($this->avatar) : 
        asset('img/default-avatar.png');
}
```

### 3. Single Responsibility Principle
Each class has a single responsibility, as seen in the separation of concerns in the observers, jobs, and controllers.

### 4. Using Dependency Injection
```php
public function handle(GitHub $github): void
{
    $user = type($this->user->fresh())->as(User::class);
    // ...
}
```

### 5. Using Form Requests for Validation
```php
final class UserAvatarUpdateRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'avatar' => ['required', 'image', 'mimes:jpg,jpeg,png', 'max:2048'],
        ];
    }
}
```

## Smart Helper Functions Usage

### 1. Collection Helper Methods
```php
collect($glob)
    ->sort()
    ->reverse()
    ->slice(4)
    ->each(fn (string $backup): bool => File::delete($backup));
```

### 2. String Helper
```php
str($this->gradient)
    ->match('/from-.*?\d{3}/')
    ->after('from-')
    ->value();
```

### 3. Type Casting Helper
```php
$user = type($request->user())->as(User::class);
```

### 4. Route Helper
```php
return to_route('profile.edit')
    ->with('flash-message', 'Avatar updated.');
```

### 5. Config Helper
```php
if (collect(config()->array('sponsors.github_usernames'))
    ->contains($this->username)) {
    return true;
}
```
