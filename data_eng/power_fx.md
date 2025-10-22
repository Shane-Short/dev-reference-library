Set(varIsSupervisor,
    !IsBlank(
        LookUp(
            Skill_Matrix_Supervisors,
            Lower(Trim(Employee_Email)) = Lower(Trim(User().Email))
                || Lower(Trim(Email)) = Lower(Trim(User().Email))
                || Lower(Trim(Title)) = Lower(Trim(User().Email)),
            true
        )
    )
);
