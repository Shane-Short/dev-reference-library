AddColumns(
    SortByColumns(
        Filter(
            Skill_Matrix_User_Settings,
            !IsBlank(Employee_Email)
        ),
        "Employee",
        Ascending
    ),
    "Title", Employee,
    "Subtitle", Employee_Email
)
