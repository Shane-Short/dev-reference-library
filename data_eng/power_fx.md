// create → clear pattern so collections exist everywhere we reference them
ClearCollect(colPresetUsersOriginal, Table({ Employee_Email:"", Employee:"" })); 
Clear(colPresetUsersOriginal);

ClearCollect(colPresetUsersWorking, Table({ Employee_Email:"", Employee:"" })); 
Clear(colPresetUsersWorking);

ClearCollect(colAddedUsers, Table({ Employee_Email:"", Employee:"" })); 
Clear(colAddedUsers);

ClearCollect(colRemovedUsers, Table({ Employee_Email:"", Employee:"" })); 
Clear(colRemovedUsers);
