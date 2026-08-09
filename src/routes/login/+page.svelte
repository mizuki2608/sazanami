<script lang="ts">
	import { signIn, signUp, authClient, emailPassword } from '$lib/auth-client';
	import { goto, invalidateAll } from '$app/navigation';

	import { page } from '$app/state';
	import { onMount } from 'svelte';
	import { ulid } from 'ulid';

	const queryParams = page.url.searchParams;

	let mode: 'login' | 'register' = $state(
		queryParams.get('mode') === 'register' ? 'register' : 'login'
	);
	let name = $state('');
	let email = $state('');
	let password = $state('');
	let error: string | null = $state(null);
	let message: string | null = $state(null);
	let isLoading = $state(false);
	let showPasswordModal = $state(false);
	let showMagicLinkModal = $state(false);
	let showForgotPasswordModal = $state(false);

	onMount(() => {
		if (queryParams.get('error') === 'invalid_token') {
			error = '無効なリンクです。';
		} else if (queryParams.get('error') === 'INVALID_TOKEN') {
			error = '無効なリンクです。';
		}
	});

	function getFormattedEmail(input: string) {
		const trimmed = input.trim();
		if (trimmed.includes('@')) {
			return trimmed;
		}
		// IDの場合はダミードメインを付与してメールアドレス化
		return `${trimmed.toLowerCase()}@sazanami.local`;
	}

	async function handleSubmit(event: Event) {
		event.preventDefault();
		if (isLoading) return;
		isLoading = true;
		error = null;
		message = null;

		const targetEmail = getFormattedEmail(email);

		try {
			if (mode === 'login') {
				console.log('Attempting login with target email:', targetEmail);
				const { data, error: apiError } = await signIn.email({
					email: targetEmail,
					password,
					callbackURL: '/home'
				});

				if (apiError) {
					console.error('Login error:', apiError);
					error = apiError.message || 'ログインに失敗しました。';
				} else if ((data as { twoFactorRedirect?: boolean }).twoFactorRedirect) {
					console.log('2FA required, redirecting to two-factor page');
					await goto(
						'/login/two-factor' + (queryParams.toString() ? '?' + queryParams.toString() : '')
					);
				} else {
					console.log('Login successful:', data);
					console.log('Redirecting to home page');
					await invalidateAll();
					await goto('/home');
				}
			} else {
				console.log('Attempting registration with target email:', targetEmail);
				const { data: signUpdata, error: signUpError } = await signUp.email({
					name: name.trim() || email.trim(),
					email: targetEmail,
					password,
					callbackURL: '/home'
				});

				if (signUpError) {
					console.error('Registration error:', signUpError);
					error = signUpError.message || '登録に失敗しました。';
				} else {
					console.log('Registration successful:', signUpdata);
					// ダミードメインの場合はメールが届かないため、自動でログインを試みる
					if (targetEmail.endsWith('@sazanami.local')) {
						console.log('Virtual domain registration, attempting auto login...');
						const { error: signInError } = await signIn.email({
							email: targetEmail,
							password,
							callbackURL: '/home'
						});

						if (signInError) {
							console.error('Auto signin error:', signInError);
							// 自動ログインに失敗した場合はログイン画面に戻す
							message = '登録が完了しました。作成したIDとパスワードでログインしてください。';
							mode = 'login';
						} else {
							await invalidateAll();
							await goto('/home');
						}
					} else {
						message =
							'確認用メールを送信しました。メールのリンクをクリックし登録を完了してください。';
						mode = 'login';
					}
				}
			}
		} catch (e: unknown) {
			console.error('Unexpected error:', e);
			if (e instanceof Error) {
				error = e.message;
			} else {
				error = '予期せぬエラーが発生しました。';
			}
		} finally {
			isLoading = false;
		}
	}

	function changeMode(newMode: 'login' | 'register') {
		mode = newMode;
		error = null;
		message = null;
		showPasswordModal = false;
		showMagicLinkModal = false;
		showForgotPasswordModal = false;
	}

	const signInWithMagicLink = async () => {
		if (isLoading) return;

		if (email === '') {
			error = 'メールアドレスを入力してください。';
			return;
		}

		if (mode === 'register' && name === '') {
			error = 'ユーザーネームを入力してください。';
			return;
		}

		isLoading = true;
		error = null;
		message = null;

		try {
			const { error: signInError } = await authClient.signIn.magicLink({
				email,
				name,
				callbackURL: '/home',
				errorCallbackURL: '/home'
			});

			if (signInError) {
				console.error('Signin error:', signInError);
				error = signInError.message || 'ログイン出来ませんでした。';
			} else {
				message = 'メールを送信しました。メールのリンクからログインしてください。';
				showMagicLinkModal = false;
			}
		} catch (e) {
			console.error('Signin error', e);
			error = '予期せぬエラーが発生しました。';
		} finally {
			isLoading = false;
		}
	};

	const signInWithGoogle = async () => {
		if (isLoading) return;
		isLoading = true;
		error = null;
		message = null;

		try {
			const { data, error: signInError } = await signIn.social({
				provider: 'google',
				callbackURL: '/home'
			});

			if (signInError) {
				console.error('Signin error:', signInError);
				error = signInError.message || 'ログイン出来ませんでした。';
			} else if ((data as { twoFactorRedirect?: boolean }).twoFactorRedirect) {
				await goto(
					'/login/two-factor' + (queryParams.toString() ? '?' + queryParams.toString() : '')
				);
			}
		} catch (e) {
			console.error('Signin error', e);
			error = '予期せぬエラーが発生しました。';
		} finally {
			isLoading = false;
		}
	};

	const forgotPassword = async () => {
		if (isLoading) return;
		if (!email) {
			error = 'メールアドレスを入力してください。';
			return;
		}

		isLoading = true;
		error = null;
		message = null;

		try {
			const { error: forgotError } = await emailPassword.forgetPassword({
				email,
				redirectTo: '/reset-password'
			});

			if (forgotError) {
				error = forgotError.message || 'パスワード再設定メールの送信に失敗しました。';
			} else {
				message = 'パスワード再設定用のメールを送信しました。';
				showForgotPasswordModal = false;
			}
		} catch (e) {
			console.error('Forgot password error:', e);
			error = '予期せぬエラーが発生しました。';
		} finally {
			isLoading = false;
		}
	};

	const signInWithPasskey = async () => {
		if (isLoading) return;
		isLoading = true;
		error = null;
		message = null;

		try {
			const { data, error: signInError } = (await signIn.passkey()) as {
				data?: { twoFactorRedirect?: boolean } | null;
				error?: { message?: string } | null;
			};
			if (signInError) {
				error =
					typeof signInError === 'string'
						? signInError
						: signInError?.message || 'パスキーによるログインに失敗しました。';
			} else if (data?.twoFactorRedirect) {
				await goto(
					'/login/two-factor' + (queryParams.toString() ? '?' + queryParams.toString() : '')
				);
			} else {
				await invalidateAll();
				await goto('/home');
			}
		} catch (e: unknown) {
			console.error('Passkey login error:', e);
			error = '予期せぬエラーが発生しました。';
		} finally {
			isLoading = false;
		}
	};

	const startAsGuest = async () => {
		if (isLoading) return;
		isLoading = true;
		error = null;
		message = null;

		try {
			const guestId = 'guest_' + ulid().toLowerCase();
			const guestPassword = ulid() + ulid(); // 十分に長くユニークなパスワード
			const guestEmail = `${guestId}@sazanami.local`;

			console.log('Registering guest user:', guestId);
			const { error: signUpError } = await signUp.email({
				name: `ゲスト (${guestId.slice(6, 12)})`,
				email: guestEmail,
				password: guestPassword,
				callbackURL: '/home'
			});

			if (signUpError) {
				console.error('Guest registration error:', signUpError);
				error = signUpError.message || 'ゲストログインに失敗しました。';
			} else {
				console.log('Guest registration successful, signing in...');
				const { error: signInError } = await signIn.email({
					email: guestEmail,
					password: guestPassword,
					callbackURL: '/home'
				});

				if (signInError) {
					console.error('Guest login error:', signInError);
					error = signInError.message || 'ゲストログインに失敗しました。';
				} else {
					await invalidateAll();
					await goto('/home');
				}
			}
		} catch (e: unknown) {
			console.error('Guest login unexpected error:', e);
			if (e instanceof Error) {
				error = e.message;
			} else {
				error = '予期せぬエラーが発生しました。';
			}
		} finally {
			isLoading = false;
		}
	};
</script>

<div class="container" style="max-width: 400px; margin: 2rem auto;">
	<div role="tablist" class="tabs tabs-bordered">
		<button
			role="tab"
			class="tab"
			class:tab-active={mode === 'login'}
			onclick={() => changeMode('login')}>ログイン</button
		>
		<button
			role="tab"
			class="tab"
			class:tab-active={mode === 'register'}
			onclick={() => changeMode('register')}>新規登録</button
		>
	</div>

	<div class="card bg-base-100 mt-4 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">
				{mode === 'login' ? 'ログイン' : '新規登録'}
			</h2>

			{#if error}
				<div role="alert" class="alert alert-error mt-4 mb-4">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-6 w-6 shrink-0 stroke-current"
						fill="none"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M10 14l2-2m0 0l2-2m-2 2l-2 2m2-2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z"
						/></svg
					>
					<span>{error}</span>
				</div>
			{/if}

			{#if message}
				<div role="alert" class="alert alert-success mt-4 mb-4">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-6 w-6 shrink-0 stroke-current"
						fill="none"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"
						/></svg
					>
					<span>{message}</span>
				</div>
			{/if}

			{#if mode === 'login'}
				<div class="card-actions mt-4">
					<button
						type="button"
						class="btn btn-secondary btn-outline w-full font-bold"
						disabled={isLoading}
						onclick={startAsGuest}
					>
						アカウント不要で始める (ゲスト)
					</button>
				</div>

				<div class="card-actions mt-4">
					<button
						type="button"
						class="btn btn-primary w-full font-bold"
						disabled={isLoading}
						onclick={signInWithPasskey}
					>
						パスキーでログイン
					</button>
				</div>
			{:else}
				<div class="card-actions mt-4">
					<button
						type="button"
						class="btn btn-secondary btn-outline w-full font-bold"
						disabled={isLoading}
						onclick={startAsGuest}
					>
						アカウント不要で始める (ゲスト)
					</button>
				</div>
			{/if}

			<div class="card-actions mt-4">
				<button
					type="button"
					class="btn w-full font-bold"
					disabled={isLoading}
					onclick={() => (showMagicLinkModal = true)}
				>
					{mode === 'login' ? 'マジックリンクでログイン' : 'マジックリンクで登録'}
				</button>
			</div>

			<div class="card-actions mt-4">
				<button
					type="button"
					class="btn w-full font-bold"
					disabled={isLoading}
					onclick={signInWithGoogle}
				>
					Googleで{mode === 'login' ? 'ログイン' : '登録'}
				</button>
			</div>

			<div class="card-actions mt-4">
				<button
					type="button"
					class="btn btn-neutral w-full font-bold"
					disabled={isLoading}
					onclick={() => (showPasswordModal = true)}
				>
					{mode === 'login' ? 'IDとパスワードでログイン' : 'IDとパスワードで登録'}
				</button>
			</div>
		</div>
	</div>
</div>

{#if showPasswordModal}
	<dialog class="modal modal-open">
		<div class="modal-box">
			<form method="dialog">
				<button
					class="btn btn-sm btn-circle btn-ghost absolute top-2 right-2"
					onclick={() => (showPasswordModal = false)}>✕</button
				>
			</form>
			<h3 class="mb-4 text-lg font-bold">
				{mode === 'login' ? 'IDとパスワードでログイン' : 'IDとパスワードで登録'}
			</h3>
			{#if error}
				<div role="alert" class="alert alert-error mb-4">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-4 w-4 shrink-0 stroke-current"
						fill="none"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M10 14l2-2m0 0l2-2m-2 2l-2 2m2-2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z"
						/></svg
					>
					<span class="text-sm">{error}</span>
				</div>
			{/if}
			{#if message}
				<div role="alert" class="alert alert-success mt-4 mb-4">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-4 w-4 shrink-0 stroke-current"
						fill="none"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"
						/></svg
					>
					<span class="text-sm">{message}</span>
				</div>
			{/if}
			<form onsubmit={handleSubmit}>
				{#if mode === 'register'}
					<div class="form-control mb-4">
						<label class="label" for="name"
							><span class="label-text">表示名（省略時はIDと同じ）</span></label
						>
						<input
							id="name"
							name="name"
							type="text"
							class="input input-bordered w-full"
							placeholder="例: たろう"
							bind:value={name}
							disabled={isLoading}
						/>
					</div>
				{/if}
				<div class="form-control mb-4">
					<label class="label" for="email"
						><span class="label-text">ユーザーID または メールアドレス</span></label
					>
					<input
						id="email"
						name="email"
						type="text"
						class="input input-bordered w-full"
						placeholder="ユーザーID または email@example.com"
						bind:value={email}
						required
						disabled={isLoading}
					/>
				</div>
				<div class="form-control mb-2">
					<label class="label" for="password"><span class="label-text">Password</span></label>
					<input
						id="password"
						name="password"
						type="password"
						class="input input-bordered w-full"
						bind:value={password}
						required
						disabled={isLoading}
					/>
				</div>
				{#if mode === 'login'}
					<div class="mb-6 flex justify-end">
						<button
							type="button"
							class="link link-primary text-xs"
							onclick={() => {
								showPasswordModal = false;
								showForgotPasswordModal = true;
							}}
						>
							パスワードを忘れた場合
						</button>
					</div>
				{:else}
					<div class="mb-6"></div>
				{/if}
				<div class="modal-action">
					<button type="submit" class="btn btn-primary w-full" disabled={isLoading}>
						{#if isLoading}<span class="loading loading-spinner"></span>{/if}
						{mode === 'login' ? 'ログイン' : '登録する'}
					</button>
				</div>
			</form>
		</div>
		<form method="dialog" class="modal-backdrop">
			<button onclick={() => (showPasswordModal = false)}>close</button>
		</form>
	</dialog>
{/if}

{#if showForgotPasswordModal}
	<dialog class="modal modal-open">
		<div class="modal-box">
			<form method="dialog">
				<button
					class="btn btn-sm btn-circle btn-ghost absolute top-2 right-2"
					onclick={() => (showForgotPasswordModal = false)}>✕</button
				>
			</form>
			<h3 class="mb-4 text-lg font-bold">パスワードを忘れた場合</h3>
			{#if error}
				<div role="alert" class="alert alert-error mb-4">
					<span class="text-sm">{error}</span>
				</div>
			{/if}
			<form
				onsubmit={(e) => {
					e.preventDefault();
					forgotPassword();
				}}
			>
				<div class="form-control mb-6">
					<label class="label" for="email_forgot"><span class="label-text">Email</span></label>
					<input
						id="email_forgot"
						name="email"
						type="email"
						class="input input-bordered w-full"
						bind:value={email}
						required
						disabled={isLoading}
					/>
				</div>
				<div class="modal-action">
					<button type="submit" class="btn btn-primary w-full" disabled={isLoading}>
						{#if isLoading}<span class="loading loading-spinner"></span>{/if}
						再設定メールを送信
					</button>
				</div>
			</form>
		</div>
		<form method="dialog" class="modal-backdrop">
			<button onclick={() => (showForgotPasswordModal = false)}>close</button>
		</form>
	</dialog>
{/if}

{#if showMagicLinkModal}
	<dialog class="modal modal-open">
		<div class="modal-box">
			<form method="dialog">
				<button
					class="btn btn-sm btn-circle btn-ghost absolute top-2 right-2"
					onclick={() => (showMagicLinkModal = false)}>✕</button
				>
			</form>
			<h3 class="mb-4 text-lg font-bold">
				{mode === 'login' ? 'マジックリンクでログイン' : 'マジックリンクで登録'}
			</h3>
			{#if error}
				<div role="alert" class="alert alert-error mb-4">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-4 w-4 shrink-0 stroke-current"
						fill="none"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M10 14l2-2m0 0l2-2m-2 2l-2 2m2-2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z"
						/></svg
					>
					<span class="text-sm">{error}</span>
				</div>
			{/if}
			{#if message}
				<div role="alert" class="alert alert-success mt-4 mb-4">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-4 w-4 shrink-0 stroke-current"
						fill="none"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"
						/></svg
					>
					<span class="text-sm">{message}</span>
				</div>
			{/if}
			<form
				onsubmit={(e) => {
					e.preventDefault();
					signInWithMagicLink();
				}}
			>
				{#if mode === 'register'}
					<div class="form-control mb-4">
						<label class="label" for="name_ml"><span class="label-text">Name</span></label>
						<input
							id="name_ml"
							name="name"
							type="text"
							class="input input-bordered w-full"
							bind:value={name}
							required
							disabled={isLoading}
						/>
					</div>
				{/if}
				<div class="form-control mb-6">
					<label class="label" for="email_ml"><span class="label-text">Email</span></label>
					<input
						id="email_ml"
						name="email"
						type="email"
						class="input input-bordered w-full"
						bind:value={email}
						required
						disabled={isLoading}
					/>
				</div>
				<div class="modal-action">
					<button type="submit" class="btn btn-primary w-full" disabled={isLoading}>
						{#if isLoading}<span class="loading loading-spinner"></span>{/if}
						{mode === 'login' ? 'メールを受け取ってログイン' : 'メールを受け取って登録'}
					</button>
				</div>
			</form>
		</div>
		<form method="dialog" class="modal-backdrop">
			<button onclick={() => (showMagicLinkModal = false)}>close</button>
		</form>
	</dialog>
{/if}
