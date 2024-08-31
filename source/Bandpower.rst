.. _built-in script bandpower:

#############################################
Band Power in EEG
#############################################

In this example, you will learn how to integrate EEG bandpower as part of your processing pipeline.
PhysioLab\ :sup:`XR` offers a built-in script to calculate the bandpower of EEG signals with a number
of parameters to configure.

.. raw:: html

    <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%; height: auto;">
        <video id="autoplay-video1" autoplay controls loop muted playsinline style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
            <source src="_static/bandpower.mp4" type="video/mp4">
            Your browser does not support the video tag.
        </video>
    </div>

Get Started
--------------------------------------------

We will get a sample EEG data. You can download the file `erp-example.p from here (~4 MB) <https://drive.google.com/file/d/1E6wYPBUIpEIQfd5Bm38Z-LkJvao7-5Tx/view?usp=sharing>`_.
Our script will run on the :ref:`replayed <feature replay>` sample data.
If you have your own EEG hardware or data, you can use that as well.

Add this bandpower script from the ``Scripting`` tab (see :ref:` this doc<feature scripting>` for more details on scripting).
You can also find this script `here <https://github.com/PhysioLabXR/PhysioLabXR-Community/blob/master/physiolabxr/scripting/Examples/Buildins/Bandpower.py>`_.

.. code-block:: python

    import numbers
    import warnings

    import numpy as np
    from scipy.interpolate import interp1d
    from scipy.signal import welch

    from physiolabxr.scripting.RenaScript import RenaScript


    class Bandpower(RenaScript):
        """Computes the PSD in short time windows,

        Parameters:
            params['source']: str, the name of the input stream to compute the psd and bandpower from. The input stream
                should include the stream with this name.
            params['window_size']: int, the size of the window in seconds to compute the PSD and bandpower. The window size
                should be smaller than the buffer duration.
            Optional: params['bands']: list of lists of two floats, the frequency range to compute the bandpower from.
                Note that because the bandpower output is set to have 5 channels, the number of bands should be less than or
                equal to 5. Additional bands will be ignored.
            Optional: params['channel_picks']: list of ints, the indices of the channels to compute the bandpower from. If None, all
                channels will be used.

        Outputs:
            outputs['psd']: [100] the power spectral density of the input signal in the specified frequency bands. The current
                implementation outputs the PSD in 100 Hz bins.
            outputs['bandpower']: [5] the bandpower of the input signal in the specified frequency bands. This implementation
                support up to 5 bands.

        """
        def __init__(self, *args, **kwargs):
            """
            Please do not edit this function
            """
            super().__init__(*args, **kwargs)

        # Start will be called once when the run button is hit.
        def init(self):
            pass

        # loop is called <Run Frequency> times per second
        def loop(self):
            assert 'source' in self.params, "Please specify the 'source' of the input stream"
            assert 'window_size' in self.params, "Please specify the param 'window_size' to compute the PSD and bandpower"
            if self.params['source'] not in self.inputs.buffer:
                warnings.warn(f'Input stream {self.params["source"]} not yet found in the buffer.')
                return
            fs = self.presets[self.params['source']].nominal_sampling_rate
            input_data, input_timestamps = self.inputs[self.params['source']]

            # skip if the data in the buffer is not enough to for a window size
            if input_data.shape[1] < (window_size := self.params['window_size'] * fs):
                return

            if 'channel_picks' in self.params:
                try:
                    input_data = input_data[self.params['channel_picks']]
                except IndexError:
                    warnings.warn('Invalid channel picks, using all channels instead.')
            # Compute PSD for all channels at once
            frequencies, psd = welch(input_data, fs, axis=1, nperseg=window_size)  # You can adjust `nperseg` based on your needs
            # take the average of the PSD across all channels
            avg_psd = np.mean(psd, axis=0)
            # reduce to or pad to at most 100 Hz bins for outputting the psd

            new_freq_range = np.linspace(1, 100, 100)
            interp_func = interp1d(frequencies, avg_psd, kind='linear', fill_value='extrapolate')
            avg_psd_interpolated = interp_func(new_freq_range)

            self.outputs['psd'] = avg_psd_interpolated

            if 'bands' in self.params:
                # check if bands param is a iterable
                assert isinstance(self.params['bands'], list), "The 'bands' param should be a list"
                # check each item in the bands param is a list of two numbers
                assert all(isinstance(band, list) and len(band) == 2 and isinstance(band[0], numbers.Number)
                           and isinstance(band[1], numbers.Number) for band in self.params['bands']), \
                    "Each item in the 'bands' param should be a list of two numbers"
                # check for each band range, the first number is bigger than the second

                bandpower = []
                # compute up to 5 bands
                for band in self.params['bands'][:5]:
                    band_idx = np.where((frequencies >= band[0]) & (frequencies <= band[1]))[0]
                    bandpower.append(np.sum(avg_psd[band_idx]))
                # pad to 5 bands if less than 5 bands are specified
                bandpower += [0] * (5 - len(bandpower))
                # normalize the bandpower
                bandpower = np.array(bandpower) / np.sum(bandpower)
                self.outputs['bandpower'] = bandpower

        # cleanup is called when the stop button is hit
        def cleanup(self):
            print('Cleanup function is called')

Set the `Input Buffer Duration`, to 10 seconds, and the `Run Frequency` to 15 Hz.
We compute psd for 10 seconds of data, and we do it 15 times per second.
You can change the `Input Buffer Duration` and `Run Frequency` based on your needs.

Next, add the inputs, outputs, and params as shown below:

* Inputs:
    * **<EEG stream name>**: the stream name of the EEG data that you want to compute the bandpower from. In our example, the EEG stream from the sample data is called 'Example-BioSemi-Midline'.
* Outputs:
    * **psd**: the power spectral density of the input signal in the specified frequency bands. The number of channels should be 100, representing frequencies from 1 to 100 Hz.
    * **bandpower: the bandpower of the input signal in the specified frequency bands. The number of channels should be 5, representing the bandpower of 5 bands. You can configure the bands with the 'bands' parameter.
* Params:
    * **source**: the name of the input stream to compute the psd and bandpower from. The value of this param should be the same as <EEG stream name>. With our sample data, the value should be 'Example-BioSemi-Midline'.
    * **window_size**: the size of the window in seconds to compute the PSD and bandpower. The window size should be smaller than the buffer duration. We use 1 second in our example.
    * **bands**: list of lists of two floats, the frequency range to compute the bandpower from. Note that because the bandpower output is set to have 5 channels, the number of bands should be fewer than or equal to 5. Additional bands will be ignored. In our example, we define three bands. They are delta 0.4-4 Hz, alpha 8-13 Hz, beta 13-30 Hz.
    * **channel_picks**: list of ints, the indices of the channels to compute the bandpower from. If not provided, all channels will be used. We use all channels in our example.

The script widget after adding the script should look like this:

.. image:: media/bandpower_script_settings.png
    :width: 1080

For more details on the inputs, outputs, and params, please refer to the script docstring.

Run the script by hitting the `Run` button. You should see then add the outputs to the ``Stream`` tab.


