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
							{{ role === 'TEACHER' ? 'Оценки студентов' : 'Ваши оценки по предметам' }}
							<q-btn
								v-if="role === 'TEACHER'"
								flat
								color="primary"
								icon="add"
								label="Добавить занятие"
								@click="addNewDate"
								class="q-ml-md"
							/>
						</div>
						<div class="table-param">
							<div class="table-param-text">Группа</div>
							<q-select
								v-if="role === 'TEACHER'"
								:options="groups"
								option-label="name"
								option-value="id"
								emit-value map-options
								v-model="selectedGroup"
								dense style="min-width: 120px"
							/>
							<div class="table-param-text">Предмет</div>
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
									<q-td>{{ renderStudentName(props.row.student) }}</q-td>
									<q-td>
										<div class="q-pa-xs flex justify-center items-center">
											<q-td
												v-for="date in this.dates"
												:key="date"
											>
												<q-input
													v-model.number="localMarks[props.rowIndex]"
													type="number"
													min="2"
													max="5"
													class="grade-input"
													@blur="onGradeChange(props.rowIndex)"
													@keyup.enter="updateStudentMark(props.rowIndex)"
													dense
												/>
											</q-td>
										</div>
									</q-td>
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
		role: "TEACHER",
		groups: [],
		students: [],
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
			const flattened = []
			this.students.forEach(student => {
				this.subjects
					.filter(subject => subject.id === this.selectedSubject)
					.forEach(subject => {
						const existingMarks = Array.isArray(student.studentsMarks) ? student.studentsMarks : []
						const match = existingMarks.find(sm => sm.subject?.id === subject.id)
						flattened.push({
							student: {
								id: student.id,
								firstname: student.firstname,
								lastname: student.lastname
							},
							subject,
							marks: match?.marks || []
						})
					})
			})
			this.studentsMarks = flattened
			const dateSet = new Set()
			this.studentsMarks.forEach(row => row.marks.forEach(m => dateSet.add(m.date)))
			this.dates = Array.from(dateSet).sort()
			console.log(this.dates)
			this.localMarks = this.studentsMarks.map(row => {
				const dict = {}
				row.marks.forEach(m => dict[m.date] = m.mark)
				return dict
			})
			return this.studentsMarks
		},
		columns() {
			const dateCols = this.dates.map(dateStr => {
				const [year, month, day] = dateStr.split("-")
				return {
					name: dateStr,
					label: `${day}.${month}.${year}`,
					align: "center",
					field: dateStr,
					sortable: false,
					style: "width: 80px"
				}
			})
			const baseCol = this.role === "TEACHER"
				? {
					name: "student",
					label: "Студент",
					align: "left",
					field: row => `${row.student.firstname} ${row.student.lastname}`,
					required: true,
					style: "min-width: 200px"
				}
				: {
					name: "subject",
					label: "Предмет",
					align: "left",
					field: row => row.subject?.name || '',
					required: true,
					style: "min-width: 200px"
				}
			return [baseCol, ...dateCols]
		},
		rowKey() {
			return this.role === "TEACHER" ? "student.id" : "subject.id"
		},
	},

	methods: {
		logout() {
			window.location.href = "/login"
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
		onGradeChange(rowIndex) {
			let date = this.dates[rowIndex]
			const key = `${rowIndex}_${date}`
			this.appliedGrades[key] = true
			setTimeout(() => delete this.appliedGrades[key], 2000)
		},
		updateStudentMark(rowIndex) {
			let date = this.dates[rowIndex]
			const row = this.studentsMarks[rowIndex]
			const subjectId = row.subject?.id
			const markValue = this.localMarks[rowIndex]?.[date] ?? 0
			if (!subjectId) return
			const requestBody = {
				date: date,
				mark: markValue,
				subjectId: row.subject.id,
				studentId: row.student.id
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
		// isGradeApplied(rowIndex, date) {
		// 	return !!this.appliedGrades[`${rowIndex}_${date}`]
		// },
		fetchSubjects() {
			axios.get("/api/v1/subjects")
				.then(res => this.subjects = res.data)
		},
		fetchStudents() {
			axios.post("/api/v1/students", {id: this.selectedGroup})
				.then(response => {
					this.students = response.data
					this.fetchMarks()
				})
		},
		fetchMarks() {
			axios.get('/api/v1/students-marks', {params: {groupId: this.selectedGroup}})
				.then(response => {
					this.studentsMarks = response.data;
					// const allDates = Array.from(new Set(
					// 	this.studentsMarks.flatMap(sm => sm.marks.map(m => m.date))
					// )).sort();
					// this.columns = [
					// 	{
					// 		name: 'student',
					// 		label: 'Студент',
					// 		field: row => row.student.fullName || row.student.lastname
					// 	},
					// 	...allDates.map(date => ({
					// 		name: date,
					// 		label: date,
					// 		field: row => row.marksByDate[date] || '',
					// 		align: 'center'
					// 	}))
					// ]
					// this.tableRows = this.studentsMarks.map(sm => ({
					// 	student: sm.student,
					// 	marksByDate: Object.fromEntries(sm.marks.map(m => [m.date, m.value]))
					// }))
				})
		}
	},

	watch: {
		selectedGroup() {
			this.fetchStudents()
		}
	},

	mounted() {
		// this.dates = Array.from(new Set(
		// 	this.studentsMarks.flatMap(student => Object.keys(student.marks))
		// )).sort();
		axios.get("/api/v1/groups")
			.then(response => {
				this.groups = response.data
			})
		this.fetchSubjects()
	}
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