// ---- Session-scoped globals ----
Set(varSelectedModuleId, Blank());
Set(varSelectedModuleName, "");

// Initialize empty collections with schema (0 rows)
ClearCollect(colRefActiveIds,            FirstN( Table({ id: "" }), 0 ));
ClearCollect(colToggleLog,               FirstN( Table({ Mod_ID: "", CatItem_ID: "", Desired: false }), 0 ));
ClearCollect(colTypeEdits,               FirstN( Table({ CatItem_ID: "", NewType: "" }), 0 ));
ClearCollect(colModuleCatItems_Working,  FirstN( Table({
    id: "",
    CatItem_ID: "",
    Category: "",
    Item: "",
    Skill_Type: "",
    IsSelectedForModule: false
}), 0 ));
