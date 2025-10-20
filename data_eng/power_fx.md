// create → clear pattern so collections exist everywhere we reference them
ClearCollect(colPresetUsersOriginal, Table({ Employee_Email:"", Employee:"" })); 
Clear(colPresetUsersOriginal);

ClearCollect(colPresetUsersWorking, Table({ Employee_Email:"", Employee:"" })); 
Clear(colPresetUsersWorking);

ClearCollect(colAddedUsers, Table({ Employee_Email:"", Employee:"" })); 
Clear(colAddedUsers);

ClearCollect(colRemovedUsers, Table({ Employee_Email:"", Employee:"" })); 
Clear(colRemovedUsers);


// Cache the selected preset
Set(varPresetName, cmbAssignPreset.Selected.Preset_Name);
Set(varPresetId,   cmbAssignPreset.Selected.Preset_ID);

// Snapshot of current users assigned to this preset
Clear(colPresetUsersOriginal);
Collect(
    colPresetUsersOriginal,
    ShowColumns(
        Filter(Skill_Matrix_User_Settings, Team_Preset_ID = varPresetId),
        "Employee_Email", "Employee"
    )
);

// Working copy (start equal to snapshot)
Clear(colPresetUsersWorking);
Collect(colPresetUsersWorking, colPresetUsersOriginal);

// Reset diffs
Clear(colAddedUsers);
Clear(colRemovedUsers);

// (optional) reset the people picker so it's ready for new adds
Reset(cmbPeople);
