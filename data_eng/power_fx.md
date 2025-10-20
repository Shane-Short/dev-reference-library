// Initialize staging collections (create → clear)

// Modules to delete
ClearCollect(colDeleteModules, Table({ Title: "" })); 
Clear(colDeleteModules);

// CatItems to delete
ClearCollect(colDeleteCatItems, Table({ CatItem_ID: "", Category: "", Item: "" })); 
Clear(colDeleteCatItems);

// Presets to delete
ClearCollect(colDeletePresets, Table({ Preset_ID: "", Preset_Name: "" })); 
Clear(colDeletePresets);

// Users to delete (if you stage users here)
ClearCollect(colDeleteUsers, Table({ email: "" })); 
Clear(colDeleteUsers);

// Remove Users from a Preset
ClearCollect(colRemoveUsersFromPreset, Table({ Employee_Email: "", Team_Preset: "", Team_Preset_ID: "" })); 
Clear(colRemoveUsersFromPreset);
