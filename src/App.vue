<template>
	<v-app>
		<h1 class="text-center">Dhomper API REST</h1>

		<v-container>
			<v-row class="mb-4" align="center">
				<!-- Search Student Textfiled -->
				<v-text-field
					v-model="searchField"
					label="Search Student"
					color="primary"
					hide-details
				/>

				<!-- Add Student Button -->
				<v-btn
					color="primary"
					@click="showFormDialog"
				>
					Add Student
				</v-btn>
			</v-row>

			<h2>Student List:</h2>
			<v-card variant="outlined">
				<!-- Loading List -->
				<div
					class="ma-4"
					v-if="isStudentListLoading && !studentsListCopy"
				>
					<h3 class="text-center">Loading...</h3>
					<div class="text-center">
						<v-progress-circular
						indeterminate
					/>
					</div>
				</div>

				<!-- List Not Found -->
				<div
					class="text-center"
					v-if="!isStudentListLoading && studentsListCopy.length == 0"
				>
					<h3 class="ma-4">Students Not Found</h3>
				</div>

				<!-- Student List -->
				<div class="my-students">
					<v-row
						v-for="student in studentsListCopy"
						:key="student.id"
						class="ma-0"
						align="center"
						justify="center"
					>
						<v-col cols="12" lg="2" md="3" sm="4" xs="2">
							<div class="photo-container">
								<v-btn
									class="floating-left"
									@click="showStudentDialog(student)"
									variant="tonal"
									icon
								>
									<v-icon size="x-large">mdi-pencil</v-icon>
								</v-btn>

								<v-btn
									class="floating-right"
									@click="showDeleteDialog(student.id)"
									variant="tonal"
									icon
								>
									<v-icon size="x-large">mdi-trash-can</v-icon>
								</v-btn>

								<img
									class="full-width"
									:src="student.imagen_thumb"
									:alt="student.nombre_completo"
								>
							</div>
						</v-col>

						<v-col class="ml-4">
							<p class="font-weight-bold">{{ student.nombre_completo }}</p>
							<p>
								<span class="font-weight-bold">Created: </span>
								{{ dateFormat(student.creado ) }}
							</p>
							<p>
								<span class="font-weight-bold">Modified: </span>
								{{ dateFormat(student.modificado) }}
							</p>
							<p>
								<span class="font-weight-bold">Grade: </span>
								{{ student.grado || "Grade Not Found" }}
							</p>
							<p>
								<span class="font-weight-bold">Group: </span>
								{{ student.grupo || "Group Not Found" }}
							</p>
							<p>
								<span class="font-weight-bold">Career: </span>
								{{ student.carrera || "Career Not Found" }}
							</p>
							<p>
								<span class="font-weight-bold">Shift: </span>
								{{ student.turno || "Shift Not Found" }}
							</p>
						</v-col>
					</v-row>
				</div>
			</v-card>
		</v-container>

		<!-- Form Dialog -->
		<v-dialog
			v-model="isFormDialogVisible"
			width="600px"
			persistent
		>
			<v-card>
				<v-card-title>
					{{ isPostMethod ? "Add a Student" : "Modify a Student" }}
				</v-card-title>

				<v-container>
					<v-form v-model="isFormValid" validate-on="blur" @submit.prevent>
						<v-text-field
							v-for="(field, index) in dialogFields"
							:key="index"
							v-model="field.model"
							:counter="field.counter"
							:label="field.label"
							:rules="field.rules"
							color="primary"
						/>

						<v-card-actions>
							<v-btn
								color="primary"
								variant="text"
								type="submit"
								@click="submitForm"
							>
								Accept
							</v-btn>
							<v-btn
								variant="text"
								@click="isFormDialogVisible = false"
							>
								Cancel
							</v-btn>
						</v-card-actions>
					</v-form>
				</v-container>
			</v-card>
		</v-dialog>

		<!-- Delete Dialog -->
		<v-dialog
			v-model="isDeleteDialogVisible"
			width="600px"
			persistent
		>
			<v-card>
				<v-card-title>
					Are you sure you want to delete this student
				</v-card-title>
				<v-card-actions>
					<v-btn
						color="primary"
						variant="text"
						@click="deleteStudent"
					>
						Accept
					</v-btn>
					<v-btn
						variant="text"
						@click="isDeleteDialogVisible = false"
					>
						Cancel
					</v-btn>
				</v-card-actions>
			</v-card>
		</v-dialog>
	</v-app>
</template>

<script setup>
import { computed, ref, onMounted } from "vue";

// Data
const MONTHS = ["Not", "Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"]

const dialogFields = ref([
	{
		label: "Name",
		counter: 50,
		model:"",
		rules: [
			value => {
				if (value.trim()) return true
				return "Name is requred."
			},
			value => {
				if (value.length <= 50) return true
				return "Name must be less than 50 characters."
			},
		]
	},
	{
		label: "Father Lastname",
		model:"",
		counter: 50,
		rules: [
			value => {
				if (value.trim()) return true
				return "Father Lastname is requred."
			},
			value => {
				if (value.length <= 50) return true
				return "Father Lastname must be less than 50 characters."
			},
		]
	},
	{
		label: "Mother Lastname",
		model:"",
		counter: 50,
		rules: [
			value => {
				if (value.length <= 50) return true
				return "Father Lastname must be less than 50 characters."
			},
		]
	},
	{
		label: "Matricula",
		model:"",
		counter: 20,
		rules: [
			value => {
				if (value.trim()) return true
				return "Matricula is requred."
			},
			value => {
				if (value.length <= 20) return true
				return "Matricula must be less than 20 characters."
			},
		]
	}
])

const studentId = ref(0)
const isPostMethod = ref(false)
const isFormDialogVisible = ref(false)
const isDeleteDialogVisible = ref(false)
const isStudentListLoading = ref(true)
const isFormValid = ref(false)
const searchField = ref("")
const studentsList = ref(null)

// Computed
const studentsListCopy = computed(() => {
	let list = []

	if(searchField.value.trim()){
		list = studentsList.value.filter(elem =>
			(" "+elem.nombre_completo).toLowerCase().indexOf((" "+searchField.value.trim()).toLowerCase()) > -1
		)
	}else {
		list = studentsList.value 
	}

	return list
})

// Methods
const dateFormat = (date) => {
	const [calendar, time] = date.split(" ")
	const [year, month, day] = calendar.split("-")

	return `${MONTHS[Number(month)]} ${day}, ${year} at ${time}`
}

const generateCurp = (fields) => {
	const [name, father, mother] = fields

	const one = father.trim().charAt(0).toUpperCase() || "X"
	const two = father.trim().substring(1).replace(/\a\e\i\o\u/gi, "").charAt(0).toUpperCase() || "X"
	const three = mother.trim().charAt(0).toUpperCase() || "X"
	const four = name.trim().charAt(0).toUpperCase() || "X"
	const [year, month, day] = new Date().toISOString().split('T')[0].split("-")
	const five = father.trim().substring(1).replace(/[aeiou]/gi, "").charAt(0).toUpperCase() || "X"
	const six = mother.trim().substring(1).replace(/[aeiou]/gi, "").charAt(0).toUpperCase() || "X"
	const seven = name.trim().substring(1).replace(/[aeiou]/gi, "").charAt(0).toUpperCase() || "X"

	const random1 = Math.floor(Math.random() * 9)
	const random2 = Math.floor(Math.random() * 9)

	return `${one}${two}${three}${four}${year.substring(2)}${month}${day}XXX${five}${six}${seven}${random1}${random2}`
}

const showFormDialog = () => {
	isPostMethod.value = true
	isFormDialogVisible.value = true
	dialogFields.value.map(elem => elem.model = "")
}

const showStudentDialog = (student) => {
	studentId.value = student.id

	isPostMethod.value = false
	isFormDialogVisible.value = true

	dialogFields.value[0].model = student.nombre || ""
	dialogFields.value[1].model = student.paterno || ""
	dialogFields.value[2].model = student.materno || ""
	dialogFields.value[3].model = student.matricula || ""
}

const showDeleteDialog = (id) => {
	studentId.value = id
	isDeleteDialogVisible.value = true
}

const validateFields = () => {
	if(!dialogFields.value[0].model.trim() || dialogFields.value[0].model.length > 50) return false
	if(!dialogFields.value[1].model.trim() || dialogFields.value[0].model.length > 50) return false
	if(dialogFields.value[2].model.length > 50) return false
	if(!dialogFields.value[3].model.trim() || dialogFields.value[0].model.length > 20) return false
	return true
}

const submitForm = () => {
	isFormValid.value = validateFields()
	
	if(isFormValid.value) {
		isPostMethod.value ? createStudent() : updateStudent()
	}
}

// CRUD
const getStudentsList = async () => {
	try {
		const response = await fetch("https://webservices.mx/escolares/test/alumnos/listar");
		const data = await response.json();
		return data.response;
	} catch (error) {
		return console.log(error);
	}
}

const createStudent = async () => {
	const fields = dialogFields.value.map(elem => elem.model.trim())
	const [name, father, mother, matricula] = fields

	await fetch("https://webservices.mx/escolares/test/alumnos/agregar", {
		method: "POST",
		headers: {
			"Content-Type" : "application/json"
		},
		mode: "no-cors",
		body: JSON.stringify({
			clave: generateCurp(fields),
			matricula: matricula,
			paterno: father,
			materno: mother,
			nombre: name
		})
	})
	.then(response => response.json)
	.then(() => isFormDialogVisible.value = false)
	.catch(error => console.log(error))

	studentsList.value = await getStudentsList()
}

const updateStudent = async () => {
	const fields = dialogFields.value.map(elem => elem.model.trim())
	const [name, father, mother, matricula] = fields

	await fetch(`https://webservices.mx/escolares/test/alumnos/guardar/${studentId.value}`, {
		method: "POST",
		mode: "no-cors",
		headers: {
			"Content-type": "application/json",
		},
		body: JSON.stringify({
			id: studentId.value,
			clave: generateCurp(fields),
			matricula: matricula,
			paterno: father,
			materno: mother,
			nombre: name
		})
	})
	.then(() => isFormDialogVisible.value = false)
	.catch(error => console.log(error))

	studentsList.value = await getStudentsList()
}

const deleteStudent = async () => {
	await fetch(`https://webservices.mx/escolares/test/alumnos/eliminar/${studentId.value}`, {
		method: "DELETE",
	})
	.then(() => isDeleteDialogVisible.value = false)
	.then(res => res.json)
	.catch(error => console.log(error))

	studentsList.value = studentsList.value.filter(
		elem => elem.id !== studentId.value
	)
}

// Mounted
onMounted(async () => {
	studentsList.value = await getStudentsList()
	isStudentListLoading.value = false
})

</script>

<style scoped>
.photo-container {
	position: relative;
	text-align: center;
}
.floating-left{
	position: absolute;
	top: 8px;
	left: 8px;
}
.floating-right {
	position: absolute;
	top: 8px;
	right: 8px;
}
.full-width {
	width: 100%
}
.my-students >*:nth-child(odd) {
	background-color: #BBB;
}
.my-students >*:nth-child(even) {
	background-color: #CCC;
}

@media (max-width: 600px) {
	.full-width {
		width: 75%
	}
}
</style>