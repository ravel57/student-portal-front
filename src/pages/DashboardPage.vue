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
						<q-table :rows="tableRows" :columns="columns" :row-key="rowKey" flat bordered hide-bottom>
							<template v-slot:body="props">
								<q-tr>
									<q-td>{{ renderStudentName(props.row.student) }}</q-td>
									<q-td>
										<div class="q-pa-xs flex justify-center items-center">
											<q-input
												v-model.number="localMarks[props.rowIndex][date]"
												type="number"
												min="2"
												max="5"
												class="grade-input"
												@blur="onGradeChange(props.rowIndex, date)"
												@keyup.enter="onGradeEnter(props.rowIndex, date)"
												dense
											/>
											<q-icon v-if="isGradeApplied(props.rowIndex, date)" name="check"
													color="positive" size="20px" class="q-ml-xs"/>
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
import axios from 'axios';

export default {
	name: "DashboardPage",

	data: () => ({
		role: "TEACHER",
		groups: [],
		studentsMarks: [],
		localMarks: [],
		appliedGrades: {},
		dates: [],
		subjects: [],
		selectedSubject: "",
		selectedGroup: ""
	}),

	computed: {
		tableRows() {
			return this.studentsMarks;
		},
		columns() {
			const dateCols = this.dates.map(dateStr => {
				const [year, month, day] = dateStr.split("-");
				return {
					name: dateStr,
					label: `${day}.${month}`,
					align: "center",
					field: dateStr,
					sortable: false,
					style: "width: 80px"
				};
			});
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
				};
			return [baseCol, ...dateCols];
		},
		rowKey() {
			return this.role === "TEACHER" ? "student.id" : "subject.id";
		}
	},

	methods: {
		logout() {
			window.location.href = "/login";
		},
		renderStudentName(student) {
			return `${student.firstname} ${student.lastname}`;
		},
		addNewDate() {
			const dateStr = new Date().toISOString().split('T')[0];
			if (!this.dates.includes(dateStr)) {
				this.dates.push(dateStr);
				this.dates.sort();
				this.studentsMarks.forEach((row, i) => {
					this.localMarks[i] = this.localMarks[i] || {};
					this.localMarks[i][dateStr] = 0;
				});
			}
		},
		onGradeChange(rowIndex, date) {
			const key = `${rowIndex}_${date}`;
			this.appliedGrades[key] = true;
			setTimeout(() => delete this.appliedGrades[key], 2000);
		},
		onGradeEnter(rowIndex, date) {
			const row = this.studentsMarks[rowIndex];
			const studentId = row.student.id;
			const subjectId = row.subject?.id;
			const markValue = this.localMarks[rowIndex]?.[date] ?? 0;
			if (!subjectId) return;

			axios.post("/api/v1/update-mark", {
				student: `${row.student.firstname} ${row.student.lastname}`,
				date,
				mark: markValue,
				subject: row.subject.name,
				studentId
			}).then(() => {
				const existing = row.marks.find(m => m.date === date);
				if (existing) existing.mark = markValue;
				else row.marks.push({date, mark: markValue});
				if (!this.dates.includes(date)) this.dates.push(date);
				this.onGradeChange(rowIndex, date);
			});
		},
		isGradeApplied(rowIndex, date) {
			return !!this.appliedGrades[`${rowIndex}_${date}`];
		},
		fetchSubjects() {
			axios.get("/api/v1/subjects").then(res => this.subjects = res.data);
		},
		fetchData() {
			console.log(this.selectedGroup)
			console.log(this.groups)
			console.log(this.groups.find(group => {
				return group.id === this.selectedGroup
			}))
			axios.post("/api/v1/students", {id: this.groups.find(group => group.id === this.selectedGroup).id}
			).then(res => {
				const students = res.data;
				const flattened = [];
				students.forEach(student => {
					this.subjects.forEach(subject => {
						const existingMarks = Array.isArray(student.studentsMarks) ? student.studentsMarks : [];
						const match = existingMarks.find(sm => sm.subject?.id === subject.id);
						flattened.push({
							student: {id: student.id, firstname: student.firstname, lastname: student.lastname},
							subject,
							marks: match?.marks || []
						});
					});
				});
				this.studentsMarks = flattened;
				this.initializeDatesAndLocalMarks();
			});
		},
		initializeDatesAndLocalMarks() {
			const dateSet = new Set();
			this.studentsMarks.forEach(row => row.marks.forEach(m => dateSet.add(m.date)));
			this.dates = Array.from(dateSet).sort();
			this.localMarks = this.studentsMarks.map(row => {
				const dict = {};
				row.marks.forEach(m => dict[m.date] = m.mark);
				return dict;
			});
		}
	},

	watch: {
		selectedGroup() {
			this.fetchData()
		}
	},

	mounted() {
		axios.get("/api/v1/groups").then(res => this.groups = res.data);
		this.fetchSubjects();
	}
};
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