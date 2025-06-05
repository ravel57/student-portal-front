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
							<!-- Кнопка «Добавить занятие» появляется только у преподавателя -->
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

						<q-table
							:rows="tableRows"
							:columns="columns"
							:row-key="rowKey"
							flat
							bordered
							hide-bottom
						>
							<!-- Ячейка «Студент» -->
							<template v-slot:body-cell-student="props">
								{{ props.row.student.firstname }} {{ props.row.student.lastname }}
							</template>

							<!-- Ячейка «Предмет» -->
							<template v-slot:body-cell-subject="props">
								<q-select
									v-if="role === 'TEACHER'"
									:options="subjects"
									option-label="name"
									option-value="id"
									emit-value
									map-options
									v-model="props.row.subject.id"
									@update:model-value="val => onSubjectChange(props.rowIndex, val)"
									dense
									style="min-width: 120px"
								/>
								<span v-else>
									{{ props.row.subject ? props.row.subject.name : '' }}
								</span>
							</template>

							<!-- Динамические ячейки для каждой даты (даты хранятся в this.dates) -->
							<template
								v-for="date in dates"
								v-slot:[`body-cell-${date}`]="props"
							>
								<div class="q-pa-xs flex justify-center items-center">
									<!-- Поле для ввода оценки -->
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
									<!-- Галочка, если оценка применена -->
									<q-icon
										v-if="isGradeApplied(props.rowIndex, date)"
										name="check"
										color="positive"
										size="20px"
										class="q-ml-xs"
									/>
								</div>
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
		role: "TEACHER", //	ADMIN TEACHER STUDENT
		groups: [],
		students: [],
		studentsMarks: [],
		localMarks: [],
		appliedGrades: {},
		dates: [],
		subjects: []
	}),

	computed: {
		tableRows() {
			console.log('tableRows:', this.studentsMarks);
			return this.studentsMarks;
		},
		columns() {
			console.log('columns studentsMarks:', this.studentsMarks);
			const dateCols = this.dates.map(dateStr => {
				if (typeof dateStr !== 'string' || !dateStr.includes('-')) {
					console.log('Некорректная дата:', dateStr);
					return null;
				}
				const [year, month, day] = dateStr.split("-");
				console.log(5)
				const label = `${day}.${month}`;
				console.log(6)
				return {
					name: dateStr,
					label,
					align: "center",
					field: dateStr,
					sortable: false,
					style: "width: 80px"
				};
			}).filter(Boolean);

			console.log(7)
			if (this.role === "TEACHER") {
				return [
					{
						name: "subject", // Добавляем колонку Предмет для преподавателя
						label: "Предмет",
						align: "left",
						field: row => {
							console.log('columns subject field row:', row);
							return row.subject ? row.subject.name : '';
						},
						required: true,
						style: "min-width: 200px"
					},
					{
						name: "student",
						label: "Студент",
						align: "left",
						field: row => {
							console.log('columns field row:', row);
							// Проверяем, что student существует
							if (row && row.student) {
								return `${row.student.firstname} ${row.student.lastname}`;
							}
							console.log('columns field row.student is undefined:', row);
							return ''; // Возвращаем пустую строку, если student не существует
						},
						required: true,
						style: "min-width: 200px"
					},
					...dateCols
				];
			} else {
				return [
					{
						name: "subject",
						label: "Предмет",
						align: "left",
						field: row => row.subject ? row.subject.name : '',
						required: true,
						style: "min-width: 200px"
					},
					...dateCols
				];
			}
		},
		rowKey() {
			return this.role === "TEACHER" ? "student.id" : "subject.id";
		}
	},

	methods: {
		logout() {
			window.location.href = "/login";
		},

		addNewDate() {
			const today = new Date();
			const dateStr = today.toISOString().split('T')[0]; // Формат YYYY-MM-DD

			if (this.dates.includes(dateStr)) {
				console.log('Занятие на эту дату уже существует');
				return;
			}

			// Если нет предметов, не можем добавить занятие
			if (this.subjects.length === 0) {
				console.log('Нет доступных предметов');
				return;
			}

			this.dates.push(dateStr);
			this.dates.sort();

			// Добавляем пустые оценки для всех студентов
			this.studentsMarks.forEach((row, index) => {
				if (!this.localMarks[index]) {
					this.localMarks[index] = {};
				}
				this.localMarks[index][dateStr] = 0;

				// Если у студента нет предмета, назначаем первый доступный
				if (!row.subject && this.subjects.length > 0) {
					row.subject = this.subjects[0];
				}
			});

			console.log('Новое занятие добавлено');
		},

		renderStudentName(student) {
			return `${student.firstname} ${student.lastname}`;
		},

		getMark(rowIndex, date) {
			const row = this.studentsMarks[rowIndex];
			const markObj = row.marks.find(m => m.date === date);
			return markObj ? markObj.mark : 0;
		},

		setMark(rowIndex, date, value) {
			if (!this.localMarks[rowIndex]) {
				this.$set(this.localMarks, rowIndex, {});
			}
			this.$set(this.localMarks[rowIndex], date, Number(value));
		},

		onGradeChange(rowIndex, date) {
			const key = `${rowIndex}_${date}`;
			this.appliedGrades[key] = true;
			setTimeout(() => {
				delete this.appliedGrades[key];
			}, 2000);
		},

		fetchSubjects() {
			axios.get("/api/v1/subjects")
				.then(response => {
					this.subjects = response.data;
					console.log('fetchSubjects: subjects loaded:', this.subjects);
				})
				.catch(e => {
					console.log('fetchSubjects error:', e);
				});
		},

		onGradeEnter(rowIndex, date) {
			const row = this.studentsMarks[rowIndex];
			console.log('onGradeEnter row:', row);
			const studentId = row.student.id;
			const subjectId = row.subject ? row.subject.id : null;
			const markValue =
				this.localMarks[rowIndex] && this.localMarks[rowIndex][date] !== undefined
					? this.localMarks[rowIndex][date]
					: this.getMark(rowIndex, date);

			if (!subjectId) {
				console.log('Нельзя поставить отметку без выбранного предмета');
				return;
			}

			const subject = this.subjects.find(s => s.id === subjectId);
			if (!subject) {
				console.log('Предмет не найден');
				return;
			}

			if (!Array.isArray(row.marks)) row.marks = [];

			const payload = {
				student: `${row.student.firstname} ${row.student.lastname}`,
				date: date,
				mark: markValue,
				subject: subject.name,
				studentId: studentId
			};

			axios
				.post("/api/v1/update-mark", payload)
				.then(() => {
					if (!row.marks) row.marks = [];
					const existing = row.marks.find(m => m.date === date);
					if (existing) {
						existing.mark = markValue;
					} else {
						row.marks.push({date: date, mark: markValue});
						if (!this.dates.includes(date)) {
							this.dates.push(date);
							this.dates.sort();
						}
					}
					this.onGradeChange(rowIndex, date);
				})
				.catch(e => {
					console.log(e);
				});
		},

		isGradeApplied(rowIndex, date) {
			const key = `${rowIndex}_${date}`;
			return !!this.appliedGrades[key];
		},

		getStudentMark(rowIndex, date) {
			const row = this.studentsMarks[rowIndex];
			console.log('getStudentMark row:', row);
			if (!row) {
				console.log('getStudentMark: row is undefined', rowIndex, date);
				return 0;
			}
			if (!row.marks) {
				console.log('getStudentMark: row.marks is undefined', row);
				return 0;
			}
			const markObj = row.marks.find(m => m.date === date);
			return markObj ? markObj.mark : 0;
		},

		fetchData() {
			if (this.role === "TEACHER") {
				axios
					.get("/api/v1/students")
					.then(response => {
						const users = response.data; // List<User>
						const flattened = [];
						users.forEach(student => {
							// Проверяем существующие studentsMarks для этого студента
							const existingMarks = Array.isArray(student.studentsMarks) ? student.studentsMarks : [];
							// Для каждого доступного предмета
							this.subjects.forEach(subject => {
								// Ищем, есть ли уже запись studentsMarks для этого студента и этого предмета
								const existingEntry = existingMarks.find(sm => sm.subject && sm.subject.id === subject.id);
								if (existingEntry) {
									// Если запись существует, добавляем ее во flattened
									existingEntry.student = {
										id: student.id,
										firstname: student.firstname,
										lastname: student.lastname
									};
									if (!Array.isArray(existingEntry.marks)) {
										existingEntry.marks = [];
									}
									flattened.push(existingEntry);
								} else {
									// Если записи для этого студента и предмета нет, создаем пустую
									flattened.push({
										student: {
											id: student.id,
											firstname: student.firstname,
											lastname: student.lastname
										},
										subject: {id: subject.id, name: subject.name}, // Устанавливаем предмет
										marks: []
									});
								}
							});
						});

						this.studentsMarks = flattened;
						this.initializeDatesAndLocalMarks();

						// Добавляем дополнительные логи для отладки после flatten
						console.log('fetchData: users from backend:', users);
						console.log('fetchData: flattened:', flattened);
						console.log('fetchData: studentsMarks after flatten:', this.studentsMarks);

						this.studentsMarks.forEach((row, i) => {
							if (!row.student || !row.marks || !row.subject || row.subject.id === undefined) {
								console.log('fetchData: BAD ROW after flatten', i, row);
							}
						});

					})
					.catch(e => {
						console.log('fetchData error:', e);
					});
			} else if (this.role === "STUDENT") {
				axios
					.get("/api/v1/students-marks")
					.then(response => {
						this.studentsMarks = response.data.map(row => {
							// Гарантируем, что subject является объектом с id
							let subject = row.subject;
							if (!subject || typeof subject !== 'object') {
								subject = {id: null, name: ''};
							} else if (subject.id === undefined) {
								subject.id = null;
							}
							// Гарантируем, что marks является массивом
							const marks = Array.isArray(row.marks) ? row.marks : [];

							return {...row, subject, marks};
						});
						this.initializeDatesAndLocalMarks();

						// Добавляем дополнительные логи для студента
						console.log('fetchData: students-marks loaded:', this.studentsMarks);
						this.studentsMarks.forEach((row, i) => {
							if (!row.student || !row.marks || !row.subject || row.subject.id === undefined) {
								console.log('fetchData (student): BAD ROW after processing', i, row);
							}
						});

					})
					.catch(() => {
						console.log('fetchData error: Не удалось загрузить ваши оценки');
					});
			}
		},

		initializeDatesAndLocalMarks() {
			console.log(1)
			const dateSet = new Set();
			this.studentsMarks.forEach(row => {
				// Гарантируем, что marks является массивом перед итерацией
				if (Array.isArray(row.marks)) {
					row.marks.forEach(m => {
						dateSet.add(m.date);
					});
				}
			});
			console.log(2)
			this.dates = Array.from(dateSet).filter(d => typeof d === 'string' && d.includes('-')).sort();
			console.log('studentsMarks', this.studentsMarks)
			this.localMarks = this.studentsMarks.map(row => {
				const dict = {};
				// Гарантируем, что marks является массивом перед итерацией
				if (Array.isArray(row.marks)) {
					row.marks.forEach(m => {
						dict[m.date] = m.mark;
					});
				}
				return dict;
			});
		},

		logRow(row, col) {
			console.log('row:', row, 'col:', col);
			return true; // всегда true, чтобы не мешать выводу
		},

		onSubjectChange(rowIndex, subjectId) {
			console.log('onSubjectChange', rowIndex, subjectId);
			const subject = this.subjects.find(s => s.id === subjectId);
			if (subject) {
				// Присваиваем весь объект subject, а не только ID
				this.$set ? this.$set(this.studentsMarks[rowIndex], 'subject', subject) : this.studentsMarks[rowIndex].subject = subject;
			} else {
				// Если предмет не найден (например, выбрали пустое значение, если оно есть), сбросим subject
				this.$set ? this.$set(this.studentsMarks[rowIndex], 'subject', {
					id: null,
					name: ''
				}) : this.studentsMarks[rowIndex].subject = {id: null, name: ''};
			}
		}
	},

	mounted() {
		this.fetchData();
		this.fetchSubjects(); // Добавляем загрузку предметов при монтировании компонента
	}
};
</script>

<style scoped>
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

/* Добавляем стиль для жирных заголовков таблицы */
:deep(.q-table th) {
	font-weight: bold;
}
</style>
