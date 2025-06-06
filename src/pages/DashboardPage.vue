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
				<q-card>
					<q-card-section>
						<div class="text-h6 q-mb-md flex items-center">
							{{ title }}
							<q-btn
								v-if="role === 'ADMIN'"
								flat
								color="primary"
								icon="add"
								label="Добавить занятие"
								@click="addNewDate"
								class="q-ml-md"
							/>
						</div>
						<div class="table-param">
							<div
								v-if="role === 'TEACHER'"
								class="table-param-text"
								v-text="'Группа'"
							/>
							<q-select
								v-if="role === 'TEACHER'"
								:options="groups"
								option-label="name"
								option-value="id"
								emit-value map-options
								v-model="selectedGroup"
								dense style="min-width: 120px"
							/>
							<div
								v-if="role === 'TEACHER'"
								class="table-param-text"
								v-text="'Предмет'"
							/>
							<q-select
								v-if="role === 'TEACHER'"
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
						<q-table
							:rows="tableRows"
							:columns="columns"
							:row-key="rowKey"
							flat
							bordered
							hide-bottom
						>
							<template v-slot:body="props">
								<q-tr>
									<q-td v-if="this.role === 'TEACHER'">
										{{ renderStudentName(props.row.student) }}
									</q-td>
									<q-td v-else>
										{{ props.row.subject.name }}
									</q-td>
									<template v-if="this.role === 'TEACHER'">
										<q-td v-for="subject in subjects" :key="subject.id" class="text-center">
											<div v-for="date in dates" :key="date" class="q-mb-xs">
												<q-input
													v-model.number="localMarks[props.rowIndex][subject.id][dateMap[date]]"
													type="number"
													min="2"
													max="5"
													class="grade-input"
													@blur="onGradeChange(props.rowIndex, subject.id, dateMap[date])"
													@keyup.enter="updateStudentMark(props.rowIndex, subject.id, dateMap[date])"
													dense
												/>
											</div>
										</q-td>
									</template>
									<template v-else>
										<q-td v-for="date in dates" :key="date" class="text-center">
											<q-input
												v-model.number="localMarks[props.rowIndex][dateMap[date]]"
												type="number"
												min="2"
												max="5"
												class="grade-input"
												@blur="onGradeChange(props.rowIndex, dateMap[date])"
												@keyup.enter="updateStudentMark(props.rowIndex, dateMap[date])"
												dense
											/>
										</q-td>
									</template>
								</q-tr>
							</template>
						</q-table>
					</q-card-section>
				</q-card>
			</q-page>
		</q-page-container>
	</q-layout>
</template>

<script>
import axios from 'axios'

export default {
	name: "DashboardPage",

	data: () => ({
		role: null,
		groups: [],
		students: [],
		student: null,
		studentsMarks: [],
		localMarks: [],
		appliedGrades: {},
		dates: [],
		subjects: [],
		selectedSubject: "",
		selectedGroup: null
	}),

	computed: {
		tableRows() {
			const isTeacher = Array.isArray(this.students);
			const dateSet = new Set();
			const subjectMap = {};
			this.subjects.forEach(sub => subjectMap[sub.id] = sub);
			let rows = [];
			if (isTeacher) {
				this.students.forEach(student => {
					const allMarks = (student.studentsMarks || []).flatMap(sm => sm.marks || []);
					allMarks.forEach(m => dateSet.add(m.date));
					const marksBySubjectAndDate = {};
					allMarks.forEach(m => {
						const subjId = m.subject.id;
						if (!marksBySubjectAndDate[subjId]) marksBySubjectAndDate[subjId] = {};
						marksBySubjectAndDate[subjId][m.date] = m.value;
					});
					rows.push({
						student,
						marksBySubjectAndDate,
					});
				});
			} else {
				this.student = this.student || {}
				const allMarks = (this.student.studentsMarks || []).flatMap(sm => sm.marks || [])
				rows = []
				this.subjects.forEach(subject => {
					const marksByDate = {}
					allMarks
						.filter(m => m.subject.id === subject.id)
						.forEach(m => marksByDate[this.formatDate(m.date)] = m.value)
					rows.push({
						subject,
						marksByDate
					})
				})
			}
			const rawDates = Array.from(dateSet).sort();
			const formattedDates = rawDates.map(this.formatDate);
			this.dateMap = Object.fromEntries(formattedDates.map((f, i) => [f, rawDates[i]]));
			this.dates = formattedDates;
			this.localMarks = rows.map(row => isTeacher ? row.marksBySubjectAndDate : row.marksByDate);
			return rows;
		},
		columns() {
			if (this.role === 'TEACHER') {
				return [
					{
						name: 'student',
						label: 'Студент',
						field: row => `${row.student.lastname} ${row.student.firstname}`,
					},
					...this.subjects.map(subject => ({
						name: subject.name,
						label: subject.name,
						align: 'center',
						// Показ первой оценки по каждому предмету в первую дату (можно усложнить)
						field: row => this.dates.map(date =>
							row.marksBySubjectAndDate[subject.id][this.dateMap[date]] || ''
						)
					}))
				];
			} else {
				return [
					{
						name: 'subject',
						label: 'Предмет',
						field: row => row.subject.name,
					},
					...this.dates.map(date => ({
						name: date,
						label: date,
						align: 'center',
						field: row => row.marksByDate[this.dateMap[date]] || '',
					}))
				];
			}
		},
		rowKey() {
			return this.role === "TEACHER" ? "student.id" : "subject.id"
		},
		title() {
			switch (this.role) {
				case "ADMIN":
					return 'Список занятий'
				case "TEACHER":
					return 'Оценки студентов'
				case "STUDENT":
					return 'Ваши оценки по предметам'
			}
		}
	},

	methods: {
		logout() {
			axios.post('/logout')
				.then(() => location.reload())
				.catch(() => {
					axios.post('/logout')
						.then(() => location.reload())
				})
		},
		renderStudentName(student) {
			return `${student.firstname} ${student.lastname}`
		},
		addNewDate() {
			const [year, month, day] = new Date().toISOString().split('T')[0].split("-")
			const dateStr = `${day}.${month}.${year}`
			if (!this.dates.includes(dateStr)) {
				this.dates.push(dateStr)
				// this.dates.sort()
				this.studentsMarks.forEach((row, i) => {
					this.localMarks[i] = this.localMarks[i] || {}
					this.localMarks[i][dateStr] = 0
				})
			}
		},
		onGradeChange(rowIndex, date) {
			const key = `${rowIndex}_${date}`
			this.appliedGrades[key] = true
			setTimeout(() => delete this.appliedGrades[key], 2000)
		},
		updateStudentMark(rowIndex, date) {
			const [year, month, day] = date.split("-")
			const row = this.studentsMarks[rowIndex]
			const markValue = this.localMarks[rowIndex][date] ?? 0
			const requestBody = {
				date: `${day}.${month}.${year}`,
				mark: markValue,
				subjectId: this.selectedSubject,
				studentId: row.student.id,
			}
			axios.post("/api/v1/update-mark", requestBody)
				.then(() => {
					const existing = row.marks.find(m => m.date === date)
					if (existing) existing.mark = markValue
					else row.marks.push({date, mark: markValue})
					if (!this.dates.includes(date)) {
						this.dates.push(date)
					}
					this.onGradeChange(rowIndex, date)
				})
		},
		fetchSubjects() {
			axios.get("/api/v1/subjects")
				.then(res => this.subjects = res.data)
		},
		fetchStudents() {
			axios.post("/api/v1/students", {id: this.selectedGroup})
				.then(response => {
					this.students = response.data
				})
		},
		formatDate(dateStr) {
			if (!dateStr) return ''
			const [year, month, day] = dateStr.split('-')
			return `${day}.${month}.${year}`
		},
	},

	watch: {
		selectedGroup() {
			this.fetchStudents()
		}
	},

	mounted() {
		axios.get("/api/v1/me")
			.then(response => {
				this.role = response.data.role
				if (this.role === 'TEACHER' || this.role === 'ADMIN') {
					axios.get("/api/v1/groups")
						.then(response => {
							this.groups = response.data
						})
				} else if (this.role === 'STUDENT') {
					this.students = response.data
				}
				this.fetchSubjects()
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