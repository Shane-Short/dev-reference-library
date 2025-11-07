concat(
  if(greater(length(variables('arrAddedCatIds')), 0), 'Adds completed; ', 'No adds registered; '),
  if(greater(length(variables('arrRemovedCatIds')), 0), 'Removes completed; ', 'No removes registered; '),
  if(greater(length(variables('arrUpdatedCI')), 0), 'Updates completed; ', 'No updates registered; ')
)



concat(
  coalesce(body('GET_Assignment')?['ProcessedNotes'], ''),
  outputs('CMP_NotesTail')
)



concat(
  if(
    greater(length(variables('arrAddedCatIds')),0),
    concat('Adds completed (', string(length(variables('arrAddedCatIds'))), '); '),
    'No adds registered; '
  ),
  if(
    greater(length(variables('arrRemovedCatIds')),0),
    concat('Removes completed (', string(length(variables('arrRemovedCatIds'))), '); '),
    'No removes registered; '
  ),
  if(
    greater(length(variables('arrUpdatedCI')),0),
    concat('Updates completed (', string(length(variables('arrUpdatedCI'))), '); '),
    'No updates registered; '
  )
)
