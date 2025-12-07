# Forms

## When to Use What

| Approach | Use When |
|----------|----------|
| React Hook Form + Zod | Complex forms, validation, performance |
| Server Actions | Simple forms, no client validation needed |
| Controlled inputs | Few fields, simple logic |

## React Hook Form + Zod (Recommended)

### Setup

```bash
npm install react-hook-form zod @hookform/resolvers
```

### Basic Form

```typescript
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

type FormData = z.infer<typeof schema>;

export function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: FormData) => {
    await loginUser(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          {...register('email')}
          className="w-full border rounded px-3 py-2"
        />
        {errors.email && (
          <p className="text-red-500 text-sm">{errors.email.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          {...register('password')}
          className="w-full border rounded px-3 py-2"
        />
        {errors.password && (
          <p className="text-red-500 text-sm">{errors.password.message}</p>
        )}
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="w-full bg-blue-500 text-white py-2 rounded disabled:opacity-50"
      >
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

### Complex Schema

```typescript
const registerSchema = z
  .object({
    name: z.string().min(2, 'Name must be at least 2 characters'),
    email: z.string().email('Invalid email'),
    password: z
      .string()
      .min(8, 'Password must be at least 8 characters')
      .regex(/[A-Z]/, 'Must contain uppercase letter')
      .regex(/[0-9]/, 'Must contain number'),
    confirmPassword: z.string(),
    age: z.coerce.number().min(18, 'Must be 18 or older').optional(),
    terms: z.literal(true, {
      errorMap: () => ({ message: 'You must accept the terms' }),
    }),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword'],
  });
```

### Reusable Input Component

```typescript
// components/ui/FormInput.tsx
import { UseFormRegister, FieldError } from 'react-hook-form';

interface FormInputProps {
  label: string;
  name: string;
  type?: string;
  register: UseFormRegister<any>;
  error?: FieldError;
  placeholder?: string;
}

export function FormInput({
  label,
  name,
  type = 'text',
  register,
  error,
  placeholder,
}: FormInputProps) {
  return (
    <div className="space-y-1">
      <label htmlFor={name} className="block text-sm font-medium">
        {label}
      </label>
      <input
        id={name}
        type={type}
        placeholder={placeholder}
        {...register(name)}
        className={`w-full border rounded px-3 py-2 ${
          error ? 'border-red-500' : 'border-gray-300'
        }`}
      />
      {error && <p className="text-red-500 text-sm">{error.message}</p>}
    </div>
  );
}

// Usage
<FormInput
  label="Email"
  name="email"
  type="email"
  register={register}
  error={errors.email}
/>
```

### Select and Radio

```typescript
const schema = z.object({
  role: z.enum(['admin', 'user', 'guest']),
  status: z.enum(['active', 'inactive']),
});

// Select
<select {...register('role')} className="w-full border rounded px-3 py-2">
  <option value="">Select role</option>
  <option value="admin">Admin</option>
  <option value="user">User</option>
  <option value="guest">Guest</option>
</select>

// Radio buttons
<div className="space-y-2">
  {['active', 'inactive'].map((status) => (
    <label key={status} className="flex items-center gap-2">
      <input type="radio" value={status} {...register('status')} />
      {status}
    </label>
  ))}
</div>
```

### Checkbox and Arrays

```typescript
const schema = z.object({
  interests: z.array(z.string()).min(1, 'Select at least one interest'),
  newsletter: z.boolean().default(false),
});

// Multiple checkboxes
const interests = ['tech', 'sports', 'music', 'travel'];

<div className="space-y-2">
  {interests.map((interest) => (
    <label key={interest} className="flex items-center gap-2">
      <input
        type="checkbox"
        value={interest}
        {...register('interests')}
      />
      {interest}
    </label>
  ))}
</div>

// Single checkbox
<label className="flex items-center gap-2">
  <input type="checkbox" {...register('newsletter')} />
  Subscribe to newsletter
</label>
```

### Default Values and Reset

```typescript
interface User {
  name: string;
  email: string;
}

export function EditUserForm({ user }: { user: User }) {
  const {
    register,
    handleSubmit,
    reset,
    formState: { errors, isDirty },
  } = useForm<User>({
    resolver: zodResolver(schema),
    defaultValues: user, // Pre-fill form
  });

  const onSubmit = async (data: User) => {
    await updateUser(data);
    reset(data); // Reset to new values
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* fields */}
      <button type="submit" disabled={!isDirty}>
        Save Changes
      </button>
      <button type="button" onClick={() => reset()}>
        Cancel
      </button>
    </form>
  );
}
```

### Watch and Conditional Fields

```typescript
export function PaymentForm() {
  const { register, watch } = useForm();
  const paymentMethod = watch('paymentMethod');

  return (
    <form>
      <select {...register('paymentMethod')}>
        <option value="card">Credit Card</option>
        <option value="bank">Bank Transfer</option>
      </select>

      {paymentMethod === 'card' && (
        <>
          <input {...register('cardNumber')} placeholder="Card Number" />
          <input {...register('cvv')} placeholder="CVV" />
        </>
      )}

      {paymentMethod === 'bank' && (
        <>
          <input {...register('accountNumber')} placeholder="Account Number" />
          <input {...register('routingNumber')} placeholder="Routing Number" />
        </>
      )}
    </form>
  );
}
```

### Form with API Integration

```typescript
import { useMutation } from '@tanstack/react-query';

export function CreatePostForm() {
  const { register, handleSubmit, reset, formState } = useForm<PostData>({
    resolver: zodResolver(postSchema),
  });

  const mutation = useMutation({
    mutationFn: (data: PostData) =>
      fetch('/api/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      }).then((r) => r.json()),
    onSuccess: () => {
      reset();
      // Navigate or show success
    },
  });

  return (
    <form onSubmit={handleSubmit((data) => mutation.mutate(data))}>
      <input {...register('title')} />
      <textarea {...register('content')} />

      <button
        type="submit"
        disabled={formState.isSubmitting || mutation.isPending}
      >
        {mutation.isPending ? 'Creating...' : 'Create Post'}
      </button>

      {mutation.error && (
        <p className="text-red-500">{mutation.error.message}</p>
      )}
    </form>
  );
}
```

### Server Action Form

```typescript
// app/actions.ts
'use server';

import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  message: z.string().min(10),
});

export async function submitContact(formData: FormData) {
  const data = {
    email: formData.get('email'),
    message: formData.get('message'),
  };

  const result = schema.safeParse(data);
  if (!result.success) {
    return { error: result.error.flatten().fieldErrors };
  }

  // Process form...
  return { success: true };
}

// components/ContactForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { submitContact } from '@/app/actions';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Sending...' : 'Send'}
    </button>
  );
}

export function ContactForm() {
  const [state, formAction] = useFormState(submitContact, null);

  return (
    <form action={formAction}>
      <input name="email" type="email" />
      {state?.error?.email && <p>{state.error.email}</p>}

      <textarea name="message" />
      {state?.error?.message && <p>{state.error.message}</p>}

      <SubmitButton />
      {state?.success && <p>Message sent!</p>}
    </form>
  );
}
```

## Best Practices

1. **Validate on blur** - Better UX than validate on change
2. **Show errors clearly** - Near the field, use color + text
3. **Disable submit while submitting** - Prevent double submissions
4. **Use schema for types** - `z.infer<typeof schema>` for type safety
5. **Handle server errors** - Display API validation errors
6. **Reset on success** - Clear form after successful submission
