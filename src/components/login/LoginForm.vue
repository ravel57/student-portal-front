<template>
	<q-card class="login-form">
		<q-card-section>
			<div class="text-h6">Вход в систему</div>
		</q-card-section>

		<form ref="loginForm" method="post" action="/perform_login" class="login-form">
			<q-card-section>
				<q-form @submit="onSubmit" class="q-gutter-md">
					<q-input
						v-model="email"
						label="Email"
						id="username"
						name="username"
						type="email"
						:rules="[val => !!val || 'Email обязателен', isValidEmail]"
					/>

					<q-input
						v-model="password"
						label="Пароль"
						id="password"
						type="password"
						name="password"
						:rules="[val => !!val || 'Пароль обязателен']"
					/>

					<div>
						<q-btn label="Войти" type="submit" color="primary" class="full-width"/>
					</div>
				</q-form>
			</q-card-section>
		</form>
	</q-card>
</template>

<script>
export default {
	name: 'LoginForm',
	data() {
		return {
			email: '',
			password: ''
		}
	},
	methods: {
		isValidEmail(val) {
			const emailPattern = /^(?=[a-zA-Z0-9@._%+-]{6,254}$)[a-zA-Z0-9._%+-]{1,64}@(?:[a-zA-Z0-9-]{1,63}\.){1,8}[a-zA-Z]{2,63}$/
			return emailPattern.test(val) || 'Неверный формат email'
		},
		onSubmit() {
			console.log('Form submitted:', {
				email: this.email,
				password: this.password
			})
		}
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