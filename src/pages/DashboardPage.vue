<template>
	<q-layout view="hHh lpR fFf" class="dashboard">
		<q-header elevated class="bg-primary text-white">
			<q-toolbar>
				<q-toolbar-title>Учебный портал</q-toolbar-title>
				<q-space/>
				<q-btn flat icon="logout" @click="logout"/>
			</q-toolbar>
		</q-header>
		<q-page-container>
			<q-page padding>
				<div class="table-param">
					<q-btn
						v-if="currentUserRole === 'ADMIN'"
						outline
						@click="showNewLessonModal = true"
						label="Добавить занятие"
					/>
					<q-btn
						v-if="currentUserRole === 'ADMIN'"
						outline
						@click="showNewUserModal = true"
						label="Добавить пользователя"
					/>
					<div
						v-if="currentUserRole === 'TEACHER'"
						class="table-param-text"
						v-text="'Группа'"
					/>
					<q-select
						v-if="currentUserRole === 'TEACHER'"
						:options="groups"
						option-label="name"
						option-value="id"
						emit-value map-options
						v-model="selectedGroup"
						dense style="min-width: 120px"
					/>
					<div
						v-if="currentUserRole === 'TEACHER'"
						class="table-param-text"
						v-text="'Предмет'"
					/>
					<q-select
						v-if="currentUserRole === 'TEACHER'"
						:options="subjects"
						option-label="name"
						option-value="id"
						emit-value
						map-options
						v-model="selectedSubject"
						dense
						style="min-width: 120px"
					/>
				</div>
				<div
					v-if="currentUserRole!== 'ADMIN'"
				>
					<!-- Для TEACHER обычная таблица -->
					<q-table v-if="currentUserRole !== 'STUDENT'" :rows="tableRows">
						<template v-slot:header="props">
							<q-tr :props="props">
								<q-th v-if="currentUserRole === 'TEACHER'">
									Студент
								</q-th>
								<q-th v-for="item in lessons" :key="item.date">
									{{ item.date }}
								</q-th>
								<q-th>
									Средняя оценка
								</q-th>
							</q-tr>
						</template>
						<template v-slot:body="props">
							<q-tr :props="props">
								<q-td v-if="currentUserRole === 'TEACHER'">
									{{ props.row.firstname }} {{ props.row.lastname }}
								</q-td>
								<q-td
									v-for="lesson in lessons"
									:key="lesson.date"
								>
									<q-input
										dense
										type="number"
										min="2"
										max="5"
										mask="#"
										:model-value="localMarks[props.row.id]?.[lesson.date] ?? ''"
										@update:model-value="val => onMarkInput(props.row.id, lesson.date, val)"
									/>
								</q-td>
								<q-td v-if="currentUserRole === 'TEACHER'">
									{{ getAverageMark(props.row) }}
								</q-td>
							</q-tr>
						</template>
					</q-table>

					<!-- Для STUDENT специальная таблица: строки — предметы, колонки — даты -->
					<q-table
						v-else
						:rows="studentRows"
						:columns="studentColumns"
						row-key="subject"
						flat
						bordered
						hide-bottom
					>
						<template v-slot:body="props">
							<q-tr :props="props">
								<q-td>{{ props.row.subject }}</q-td>
								<q-td
									v-for="col in studentColumns.slice(1)"
									:key="col.name"
									class="text-center"
								>
									{{ props.row[col.name] }}
								</q-td>
							</q-tr>
						</template>
					</q-table>
				</div>

				<q-dialog v-model="this.showNewLessonModal">
					<q-card>
						<q-card-section>
							Добавить занятие
						</q-card-section>
						<q-card-section>
							<q-select
								:options="groups"
								label="Группа"
								option-label="name"
								option-value="id"
								emit-value map-options
								v-model="selectedGroup"
								dense style="min-width: 120px"
							/>
							<q-select
								:options="subjects"
								label="Предмет"
								option-label="name"
								option-value="id"
								emit-value
								map-options
								v-model="selectedSubject"
								dense
								style="min-width: 120px"
							/>
							<q-input
								mask="##.##.####"
								type="date"
								v-model="this.newLessonDate"
							/>
						</q-card-section>
						<q-card-actions>
							<q-btn
								v-close-popup
								label="Отмена"
							/>
							<q-btn
								@click="this.addNewLesson"
								label="Добавить"
							/>
						</q-card-actions>
					</q-card>
				</q-dialog>

				<q-dialog v-model="this.showNewUserModal">
					<q-card>
						<q-card-section>
							Добавить пользователя
						</q-card-section>
						<q-card-section>
							<q-select
								:options="usersTypes"
								label="Тип пользователя"
								option-label="name"
								option-value="id"
								emit-value
								map-options
								v-model="newUserRole"
								dense style="min-width: 120px"
							/>
							<q-input
								label="Имя"
								v-model="this.newUserFirstname"
							/>
							<q-input
								label="Фамилия"
								v-model="this.newUserLastname"
							/>
							<q-input
								label="email"
								v-model="this.newUserEmail"
							/>
							<q-input
								label="Пароль"
								type="password"
								v-model="this.newUserPassword"
							/>
							<q-select
								:options="groups"
								label="Группа"
								option-label="name"
								option-value="id"
								emit-value
								map-options
								v-model="newUserGroupId"
								dense style="min-width: 120px"
							/>
						</q-card-section>
						<q-card-actions>
							<q-btn
								v-close-popup
								label="Отмена"
							/>
							<q-btn
								@click="this.addNewUser"
								label="Добавить"
							/>
						</q-card-actions>
					</q-card>
				</q-dialog>
			</q-page>
		</q-page-container>
	</q-layout>
</template>

<script>
import axios from 'axios'

export default {
	name: "DashboardPage",

	data: () => ({
		currentUserRole: "",
		tableRows: [],
		groups: [],
		selectedGroup: "",
		subjects: [],
		selectedSubject: "",
		students: [],
		user: {},
		showNewLessonModal: false,
		showNewUserModal: false,
		newLessonDate: "",
		lessons: [],
		localMarks: {},
		studentRows: [],
		studentColumns: [],
		allDates: [],
		studentMarksBySubject: [],
		usersTypes: ['STUDENT', 'TEACHER', 'ADMIN'],
		newUserRole: '',
		newUserFirstname: '',
		newUserLastname: '',
		newUserEmail: '',
		newUserPassword: '',
		newUserGroupId: '',
	}),

	methods: {
		logout() {
			axios.post('/logout')
				.then(() => location.reload())
				.catch(() => {
					axios.post('/logout')
						.then(() => location.reload())
				})
		},
		fetchSubjects() {
			axios.get("/api/v1/subjects")
				.then(res => this.subjects = res.data)
		},
		fetchStudents() {
			axios.post("/api/v1/students", {id: this.selectedGroup})
				.then(response => {
					this.tableRows = response.data
				})
		},
		addNewLesson() {
			axios.post("/api/v1/lesson", {
				subjectId: this.selectedSubject,
				groupId: this.selectedGroup,
				date: this.newLessonDate
			}).then(() => {
				this.showNewLessonModal = false
			})
		},
		getLessonsBy() {
			if (this.selectedSubject !== "" && this.selectedGroup !== "") {
				axios.post("/api/v1/lessons-by", {
					subjectId: this.selectedSubject,
					groupId: this.selectedGroup,
				}).then(response => {
					this.lessons = response.data
				})
			}
		},
		fillMarks() {
			this.localMarks = {};
			for (const student of this.tableRows) {
				this.localMarks[student.id] = {};
				if (student.studentsMarks && student.studentsMarks.length > 0) {
					for (const sm of student.studentsMarks) {
						for (const mark of sm.marks) {
							this.localMarks[student.id][mark.date] = mark.value;
						}
					}
				}
				for (const lesson of this.lessons) {
					if (!(lesson.date in this.localMarks[student.id])) {
						this.localMarks[student.id][lesson.date] = '';
					}
				}
			}
		},
		onMarkInput(studentId, date, value) {
			if (!this.localMarks[studentId]) this.localMarks[studentId] = {};
			this.localMarks[studentId][date] = value;
			axios.post('/api/v1/update-mark', {
				date: date,
				mark: value,
				subjectId: this.selectedSubject,
				studentId: studentId,
			})
		},
		fillStudentTable() {
			if (!this.studentMarksBySubject || this.studentMarksBySubject.length === 0) {
				this.studentRows = [];
				this.studentColumns = [];
				this.allDates = [];
				return;
			}
			const dateSet = new Set();
			this.studentMarksBySubject.forEach(item => {
				(item.marks || []).forEach(mark => {
					if (mark && mark.date) {
						dateSet.add(mark.date);
					}
				});
			});
			this.allDates = Array.from(dateSet).sort();

			this.studentColumns = [
				{name: 'subject', label: 'Предмет', align: 'left', field: 'subject'},
				...this.allDates.map(date => ({
					name: date,
					label: date,
					align: 'center',
					field: date
				}))
			];

			this.studentRows = this.studentMarksBySubject.map(subjectItem => {
				const marksByDate = {};
				(this.allDates || []).forEach(date => {
					marksByDate[date] = '';
				});
				(subjectItem.marks || []).forEach(mark => {
					marksByDate[mark.date] = mark.mark !== null && mark.mark !== undefined ? mark.mark : '';
				});
				return {
					subject: subjectItem.subject.name,
					...marksByDate
				};
			});
		},
		addNewUser() {
			axios.post("/api/v1/user", {
				firstname: this.newUserFirstname,
				lastname: this.newUserLastname,
				email: this.newUserEmail,
				password: this.newUserPassword,
				role: this.newUserRole,
				groupId: this.newUserGroupId,
			}).then(() => {
				this.showNewUserModal = false
			})
		},
		getAverageMark(row) {
			console.log(row)
			if (!row.marks || row.marks.length === 0) return '-'
			const values = row.marks
				.map(m => m.mark)
				.filter(m => typeof m === 'number' && !isNaN(m))
			if (values.length === 0) return '-'
			const sum = values.reduce((acc, v) => acc + v, 0)
			return (sum / values.length).toFixed(2)
		},
	},

	watch: {
		selectedGroup() {
			this.fetchStudents()
			if (this.currentUserRole !== 'TEACHER') return
			this.getLessonsBy()
		},
		selectedSubject() {
			this.getLessonsBy()
		},
		tableRows: {
			handler() {
				this.fillMarks();
			},
			immediate: true
		},
		lessons: {
			handler() {
				this.fillMarks();
			},
			immediate: true
		},
		studentMarksBySubject: {
			handler() {
				this.fillStudentTable();
			},
			immediate: true
		}
	},

	mounted() {
		axios.get("/api/v1/groups")
			.then(response => {
				this.groups = response.data
			})
		this.fetchSubjects()
		axios.get("/api/v1/me")
			.then(response => {
				this.user = response.data
				this.currentUserRole = this.user.role
				if (this.currentUserRole === 'STUDENT') {
					axios.get(`/api/v1/students-marks-by-subjects?studentId=${this.user.id}`).then(response => {
						this.studentMarksBySubject = response.data;
						this.fillStudentTable();
					})
				}
			})
	},
}
</script>

<style scoped>
.table-param {
	display: flex;
	gap: 10px;
	align-items: center;
}

.table-param-text {
	display: flex;
	justify-content: center;
}

.dashboard {
	min-height: 100vh;
}

.grade-input input::-webkit-outer-spin-button,
.grade-input input::-webkit-inner-spin-button {
	display: none;
	-webkit-appearance: none;
	margin: 0;
}

.grade-input input[type="number"] {
	-moz-appearance: textfield;
	text-align: center;
}

:deep(.q-table th) {
	font-weight: bold;
}
</style>