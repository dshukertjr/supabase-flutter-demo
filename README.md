# Supabase Flutter User Management

This example will set you up for a very common situation: users can sign up with a magic link and then update their account with public profile information, including a profile image.

## Technologies used

- Frontend:
  - [Flutter SDK](https://flutter.dev/) - Google's UI toolkit for crafting beautiful, natively compiled applications for mobile, web, and desktop from a single codebase.
  - [supabase_flutter](https://pub.dev/packages/supabase_flutter) for user management and image storage uploading.
- Backend:
  - [app.supabase.io](https://app.supabase.io/): hosted Postgres database with restful API for usage with supabase_flutter.

## Build from scratch

### 1. Create new project

Sign up to Supabase - [https://app.supabase.io](https://app.supabase.io) and create a new project. Wait for your database to start.

### 2. Run "User Management" Quickstart

Once your database has started, run the "User Management Starter" quickstart. Inside of your project, enter the `SQL editor` tab and scroll down until you see `User Management Starter: Set up a Public Profiles table which you can access with your API`.

### 3. Get the URL and Key

Go to the Project Settings (the cog icon), open the API tab, and find your API URL and `anon` key, you'll need these in the next step.

The `anon` key is your client-side API key. It allows "anonymous access" to your database, until the user has logged in. Once they have logged in, the keys will switch to the user's own login token. This enables row level security for your data. Read more about this [below](#postgres-row-level-security).

![image](https://user-images.githubusercontent.com/10214025/88916245-528c2680-d298-11ea-8a71-708f93e1ce4f.png)

**_NOTE_**: The `service_role` key has full access to your data, bypassing any security policies. These keys have to be kept secret and are meant to be used in server environments and never on a client or browser.

### 4. Setup deeplink redirect urls

Go to the Authentication Settings page (the user icon). Enter the flutter app redirect url below into `Additional Redirect URLs` field. Then click Save.

> io.supabase.flutterdemo://login-callback

![authentication settings page](https://user-images.githubusercontent.com/689843/124574731-f735c180-de74-11eb-8f50-2d34161261dd.png)

### 4. Run the flutter application

- Go to `lib/utils/constants.dart` file
- Update `SUPABASE_URL` and `SUPABASE_ANNON_KEY` with your URL and Key
- Run the application: `flutter run`

## Supabase details

### Postgres Row level security

This project uses very high-level Authorization using Postgres' Role Level Security.
When you start a Postgres database on Supabase, we populate it with an `auth` schema, and some helper functions.
When a user logs in, they are issued a JWT with the role `authenticated` and thier UUID.
We can use these details to provide fine-grained control over what each user can and cannot do.

This is a trimmed-down schema, with the policies:

```sql
-- Create a table for Public Profiles
create table profiles (
  id uuid references auth.users not null,
  updated_at timestamp with time zone,
  username text unique,
  avatar_url text,
  website text,

  primary key (id),
  unique(username),
  constraint username_length check (char_length(username) >= 3)
);

alter table profiles enable row level security;

create policy "Public profiles are viewable by everyone."
  on profiles for select
  using ( true );

create policy "Users can insert their own profile."
  on profiles for insert
  with check ( auth.uid() = id );

create policy "Users can update own profile."
  on profiles for update
  using ( auth.uid() = id );

-- Set up Realtime!
begin;
  drop publication if exists supabase_realtime;
  create publication supabase_realtime;
commit;
alter publication supabase_realtime add table profiles;

-- Set up Storage!
insert into storage.buckets (id, name)
values ('avatars', 'avatars');

create policy "Avatar images are publicly accessible."
  on storage.objects for select
  using ( bucket_id = 'avatars' );

create policy "Anyone can upload an avatar."
  on storage.objects for insert
  with check ( bucket_id = 'avatars' );
```

## Authors

- [Supabase](https://supabase.io)

Supabase is open source. We'd love for you to follow along and get involved at https://github.com/supabase/supabase
