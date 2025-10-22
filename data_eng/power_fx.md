AddColumns(
    SortByColumns(
        Filter(
            Skill_Matrix_User_Settings,
            !IsBlank(Employee_Email) &&
            (IsBlank(cmbDelUser.SearchText) ||
             StartsWith(Employee, cmbDelUser.SearchText) ||
             StartsWith(Employee_Email, cmbDelUser.SearchText))
        ),
        "Employee",
        Ascending
    ),
    "Title", Employee,
    "Subtitle", Employee_Email
)
