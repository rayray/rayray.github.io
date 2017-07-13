 - read/write performance on that data is terrible compared to read/write directly to the `Documents` folder
 - for large numbers of objects containing data, even "lightweight" migrations are slow
 - failing to migrate your model means your user can't access their images

If you absolutely need to store references to files in your model, **save your files/images directly 
to the app's `Documents` folder, and keep a reference to the filename.** It will save you a lot of pain
when you update the model.