## Stack two mathplot charts

```py
def stacked_charts(top_x_data, top_data_list,bottom_data_list, 
                   bottom_x_data=None, share_x_axis=True, 
                   top_config=None, bottom_config=None, 
                   top_secondary_data_list=None, top_secondary_config=None,
                   bottom_secondary_data_list=None, bottom_secondary_config=None,
                   figure_size=(10, 8), height_ratios=[2, 1], hspace=0.1,
                   save_config=None):
    """
    Creates a stacked two-panel chart.

    Args:
        top_x_data (array-like): X-axis data for the top chart.
        top_data_list (list of array-like): Datasets for the top chart.
        bottom_data_list (list of array-like): Datasets for the bottom chart.
        bottom_x_data (array-like, optional): X-axis data for the bottom chart. 
            If None, falls back to using `top_x_data`. Defaults to None.
        share_x_axis (bool, optional): If True, links the top and bottom x-axes 
            and hides the top x-axis labels. Defaults to True.
        ... [other args remain the same] ...
    """
    # If no separate bottom x-data is provided, assume they share the same x-axis
    if bottom_x_data is None:
        bottom_x_data = top_x_data

    top_config = top_config or {}
    bottom_config = bottom_config or {}
    top_secondary_config = top_secondary_config or {}
    bottom_secondary_config = bottom_secondary_config or {}

    with plt.style.context('ggplot'):
        fig, (ax1, ax2) = plt.subplots(
            2, 1, 
            figsize=figure_size, 
            sharex=share_x_axis, # <-- Now configurable
            gridspec_kw={'height_ratios': height_ratios}
        )
        fig.subplots_adjust(hspace=hspace)

        # --- 1. Top Chart ---
        one_y_axis(top_x_data, top_data_list, ax=ax1, **top_config)
        
        if top_secondary_data_list:
            ax1_twin = ax1.twinx()
            ax1_twin.grid(False) 
            one_y_axis(top_x_data, top_secondary_data_list, ax=ax1_twin, **top_secondary_config)

        # --- 2. Bottom Chart ---
        one_y_axis(bottom_x_data, bottom_data_list, ax=ax2, **bottom_config)
        
        if bottom_secondary_data_list:
            ax2_twin = ax2.twinx()
            ax2_twin.grid(False)
            one_y_axis(bottom_x_data, bottom_secondary_data_list, ax=ax2_twin, **bottom_secondary_config)

        plt.tight_layout()
        
        if save_config:
            path = save_results(save_config=save_config)
            if path:
                plt.savefig(path, dpi=300, bbox_inches='tight')
        
        plt.show()
        
        return fig, (ax1, ax2)
```
