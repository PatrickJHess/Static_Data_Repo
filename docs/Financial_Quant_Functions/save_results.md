## Pick folder and file to save

```py
def save_results(save_config: dict = None):
    """
    Interactively prompts the user to confirm and generate a safe save path.


    This function pauses execution and asks the user (y/n) if they want to save
    a file. It is environment-aware:


    - **In Google Colab:** It attempts to use '/content/drive/MyDrive'. If not
      mounted, it will try to mount it. If mounting fails, it falls back
      to the temporary '/content' directory and issues a warning.
    - **In a local environment:** It uses the current working directory.


    The function constructs a full path from the base folder and the optional
    'volume' and 'chapter' subdirectories. All path components are sanitized
    using `pathvalidate`.


    Args:
        save_config (dict, optional): A dictionary containing path components.
            'volume' (str, optional): The name of a top-level subdirectory.
            'chapter' (str, optional): The name of a nested subdirectory.
            'file_name' (str, optional): The final file name.
                Defaults to 'output.txt' if not provided or if sanitization
                results in an empty string.


    Returns:
        str or None: A complete, sanitized, absolute string path to the
            file if the user confirms 'y' and path creation succeeds.
            Returns None if the user chooses 'n' or if an error occurs.
    
    Example:
        config = {
            'volume': 'My_Notebooks',
            'chapter': 'Chapter_01',
            'file_name': 'results.json'
        }
        save_path = save_results(config)
        
        # User inputs 'y'
        
        if save_path:
            # save_path might be '/content/drive/MyDrive/My_Notebooks/Chapter_01/results.json'
            print(f"Saving to: {save_path}")
            # ... proceed to write file ...
    """



    save_config = save_config or {}


    # --- Handle IPython imports safely ---
    is_colab = 'google.colab' in sys.modules
    try:
        from IPython.display import display, Markdown as md
        def display_msg(text): display(md(text))
    except ImportError:
        def display_msg(text): print(text.replace('###', '').replace('*', ''))


    # --- 1. Get user's choice (y/n) ---
    default_value = "n"
    prompt = "❓ Do you want to save the file? (y/n) (press enter for n): "


    try:
        raw_input = input(prompt).strip().lower()
        choice = raw_input if raw_input else default_value
    except (EOFError, KeyboardInterrupt):
        return None


    while choice not in ['y', 'n']:
        choice = input(prompt).strip().lower()


    # --- 2. Handle "No" ---
    if choice == 'n':
        display_msg('### ***❌ File Not Saved***.')
        return None


    # --- 3. Handle "Yes" ---
    display_msg("### ***⌛ Generating A Path***")


    # --- Get and sanitize config values ---
    volume = sanitize_filepath(save_config.get('volume', ''))
    chapter = sanitize_filepath(save_config.get('chapter', ''))
    
    # The caller should provide the extension, but we fall back to a generic name just in case
    file_name = sanitize_filepath(save_config.get('file_name', 'output'))
    if not file_name:
        file_name = 'output'


    subfolder = os.path.join(volume, chapter)


    # --- Drive/Folder Logic ---
    base_folder = ''
    drive_path = '/content/drive/MyDrive'
    
    if is_colab:
        if os.path.exists(drive_path):
            base_folder = drive_path
        else:
            try:
                from google.colab import drive
                drive.mount('/content/drive')
                base_folder = drive_path
            except Exception as e:
                base_folder = '/content'
                display_msg(f"### ⚠️ **Drive Mount Failed:** {e}. Saving to temporary '/content' folder.")
    else:
        base_folder = os.getcwd()


    # --- Path Creation ---
    try:
        full_folder_path = os.path.join(base_folder, subfolder)
        path_obj = Path(full_folder_path)
        path_obj.mkdir(parents=True, exist_ok=True)
        
        final_path_str = os.path.join(str(path_obj), file_name)


        display_msg(f'### ✅ **File Path Generated:**\n`{final_path_str}`')
        
        if base_folder == '/content':
            display_msg('### ⚠️ *File is in a temporary location and will be lost on runtime restart.*')


        return final_path_str


    except Exception as e:
        display_msg('### ❌ **ERROR Creating Directory:**')
        print(e)
        return None

```
