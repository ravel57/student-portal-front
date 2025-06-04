<template>
	<q-card class="login-form">
		<q-card-section>
			<div class="text-h6">Вход в систему</div>
		</q-card-section>

		<q-card-section>
			<form method="post" action="/perform_login" class="q-gutter-md">
				<q-input
					id="username"
					v-model="formData.username"
					label="Email"
					type="email"
					name="username"
					:rules="[
						val => !!val || 'Email обязателен',
						val => isValidEmail(val) || 'Неверный формат email'
					]"
				/>

				<q-input
					v-model="formData.password"
					label="Пароль"
					type="password"
					name="password"
					:rules="[val => !!val || 'Пароль обязателен']"
				/>
				<input
					type="hidden"
					name="_csrf"
					:value="this.csrf"
				/>
				<q-btn
					label="Войти"
					type="submit"
					color="primary"
					class="full-width"
				/>
			</form>
		</q-card-section>
	</q-card>
</template>

<script>
export default {
	name: 'LoginForm',

	data: () => ({
		formData: {
			username: '',
			password: ''
		},
		csrf: document.querySelector('meta[name="_csrf"]').content,
	}),

	methods: {
		isValidEmail(val) {
			const emailPattern = /^(?=[a-zA-Z0-9@._%+-]{6,254}$)[a-zA-Z0-9._%+-]{1,64}@(?:[a-zA-Z0-9-]{1,63}\.){1,8}[a-zA-Z]{2,63}$/
			return emailPattern.test(val)
		},
	}
}
</script>

<style scoped>
.login-form {
	max-width: 400px;
	margin: 0 auto;
	padding: 20px;
}
</style>