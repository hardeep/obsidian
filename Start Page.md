## Tasks from Journal

```dataviewjs
const folder = "Daily";

dv.taskList(
  dv.pages(`"${folder}"`)
    .file
    .tasks
    .where(t => !t.completed),
  true
);
```

