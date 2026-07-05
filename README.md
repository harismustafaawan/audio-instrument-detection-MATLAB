# Audio Signal Processing and Instrument Identification (MATLAB GUI)

A MATLAB App Designer-based GUI for processing audio signals and identifying musical instruments (drums, guitar, piano) using frequency domain analysis (FFT) — without any AI/ML.

## Features
- Fade In / Fade Out, Volume, and Speed control
- Instrument detection using FFT-based frequency analysis
- Time-domain and frequency-domain signal visualization
- Low-pass, High-pass, Band-pass, and Band-stop filters

## Concepts Used
- Fourier Transform (FFT)
- Signal Processing
- MATLAB App Designer (GUI Development)
- Digital Filters
## Code
classdef appfinal < matlab.apps.AppBase

    % Properties that correspond to app components
    properties (Access = public)
        UIFigure           matlab.ui.Figure
        Lamp               matlab.ui.control.Lamp
        Label_2            matlab.ui.control.Label
        InstrumentLabel    matlab.ui.control.Label
        BandStopButton     matlab.ui.control.Button
        BandStartButton    matlab.ui.control.Button
        LowPassButton      matlab.ui.control.Button
        HighPassButton     matlab.ui.control.Button
        AddFFTButton       matlab.ui.control.Button
        SpeedSlider        matlab.ui.control.Slider
        SpeedSliderLabel   matlab.ui.control.Label
        VolumeSlider       matlab.ui.control.Slider
        Label              matlab.ui.control.Label
        FadeOutButton      matlab.ui.control.Button
        FadeInButton       matlab.ui.control.Button
        StartStopButton    matlab.ui.control.Button
        UploadAudioButton  matlab.ui.control.Button
        UIAxes3            matlab.ui.control.UIAxes
        UIAxes2            matlab.ui.control.UIAxes
        UIAxes             matlab.ui.control.UIAxes
    end

    % Callbacks that handle component events
    methods (Access = private)

        % Button pushed function: UploadAudioButton
        function UploadAudioButtonPushed(app, event)
            [filename, pathname] = uigetfile({'*.wav'}, 'Select an audio file');
if isequal(filename, 0)
    return;
end

fullFilePath = fullfile(pathname, filename);
[app.AudioData, app.Fs] = audioread(fullFilePath);
t = (0:length(app.AudioData)-1) / app.Fs;
plot(app.UIAxes, t, app.AudioData(:,1));
app.OriginalAudioData = app.AudioData;

if size(app.AudioData, 2) >= 1
    audio = app.AudioData(:,1);
else
    audio = app.AudioData;
end

if isempty(audio)
    uialert(app.UIFigure, 'Loaded audio is empty!', 'Error');
    return;
end

n = length(audio);
if n < 2
    uialert(app.UIFigure, 'Audio too short for analysis!', 'Error');
    return;
end

zcr = sum(abs(diff(sign(audio)))) / length(audio);

nfft = 2^nextpow2(n);
Y = abs(fft(audio, nfft));
f = (0:nfft-1)*(app.Fs/nfft);
Y(1) = 0;

[~, peakIdx] = max(Y);
fundamentalFreq = f(peakIdx);

if fundamentalFreq < 200 && zcr > 0.1
    instrument = 'Drum';
elseif fundamentalFreq > 200 && fundamentalFreq < 1000
    instrument = 'Guitar';
elseif fundamentalFreq > 1000
    instrument = 'Piano';
else
    instrument = 'Unknown';
end

uialert(app.UIFigure, [instrument ' detected from audio analysis.'], 'Instrument Detection');

if isprop(app, 'InstrumentLabel')
    app.InstrumentLabel.Text = ['Instrument: ' instrument];
end
if isprop(app, 'InstrumentLamp')
    app.InstrumentLamp.Color = [1 0.5 0];
end

        end

        % Button pushed function: StartStopButton
        function StartStopButtonPushed(app, event)
            if isempty(app.AudioData) || isempty(app.Fs)
    uialert(app.UIFigure, 'Audio not loaded. Please upload an audio file first.', 'Error');
    return;
end

if isempty(app.Player) || ~isvalid(app.Player)
    app.Player = audioplayer(app.AudioData, app.Fs);
end

if app.IsPlaying
    stop(app.Player);
    app.StartStopButton_2.Text = 'Start';
    app.Lamp.Color = [0.5 0.5 0.5];
else
    play(app.Player);
    app.StartStopButton_2.Text = 'Stop';
    app.Lamp.Color = [1 0.5 0];
end

app.IsPlaying = ~app.IsPlaying;

        end

        % Button pushed function: FadeInButton
        function FadeInButtonPushed(app, event)
                  y = app.AudioData;
n = size(y, 1);                     % number of samples
fade = linspace(0, 1, round(n*0.2))';  % fade vector
fade = repmat(fade, 1, size(y, 2));     % expand for all channels
y(1:length(fade), :) = y(1:length(fade), :) .* fade;

app.AudioData = y;
plot(app.UIAxes, y(:,1));


        end

        % Button pushed function: FadeOutButton
        function FadeOutButtonPushed(app, event)
                     y = app.AudioData;
n = size(y,1);
fade = repmat(linspace(1,0,round(n*0.2))', 1, size(y,2));
y(end-length(fade)+1:end, :) = y(end-length(fade)+1:end, :) .* fade;

app.AudioData = y;
plot(app.UIAxes, y(:,1));

        end

        % Button pushed function: AddFFTButton
        function AddFFTButtonPushed(app, event)
                        y = app.AudioData;
Y = abs(fft(y));
f = linspace(0, app.Fs, length(Y));
plot(app.UIAxes2, f, Y); % Directly plot on UIA 

        end

        % Value changed function: VolumeSlider
        function VolumeSliderValueChanged(app, event)
            value = app.VolumeSlider.Value;
                if isempty(app.OriginalAudioData)
        uialert(app.UIFigure, 'Please upload audio first.', 'Error');
        return;
    end

    volumeFactor = app.VolumeSlider.Value / 100;
    newAudio = app.OriginalAudioData * volumeFactor;

    % If audio is currently playing, remember position
    wasPlaying = app.IsPlaying;
    currentSample = 1;

    if wasPlaying
        currentSample = get(app.Player, 'CurrentSample');
        stop(app.Player);
    end

    % Update player with new volume
    app.AudioData = newAudio;
    app.Player = audioplayer(app.AudioData, app.Fs);

    % Resume from last position
    if wasPlaying
        play(app.Player, currentSample);
    end

        end

        % Value changed function: SpeedSlider
        function SpeedSliderValueChanged(app, event)
            value = app.SpeedSlider.Value;
                if isempty(app.OriginalAudioData)
        uialert(app.UIFigure, 'Please upload audio first.', 'Error');
        return;
    end

    speedFactor = app.SpeedSlider.Value / 50;  % Normal speed = 50

    if speedFactor <= 0
        uialert(app.UIFigure, 'Speed must be greater than zero.', 'Error');
        return;
    end

    y = app.OriginalAudioData;
    t_original = linspace(0, 1, length(y));
    t_new = linspace(0, 1, round(length(y)/speedFactor));

    % Interpolation for mono/stereo
    if size(y,2) == 1
        newAudio = interp1(t_original, y, t_new)';
    else
        newAudio = [interp1(t_original, y(:,1), t_new)', ...
                    interp1(t_original, y(:,2), t_new)'];
    end

    % Stop previous player if needed
    wasPlaying = app.IsPlaying;
    if wasPlaying
        stop(app.Player);
    end

    app.AudioData = newAudio;
    app.Player = audioplayer(app.AudioData, app.Fs);

    if wasPlaying
        play(app.Player);
    end



        end

        % Button pushed function: HighPassButton
        function HighPassButtonPushed(app, event)
                       
 if isempty(app.AudioData)
    uialert(app.UIFigure, 'Please upload audio first.', 'Error');
    return;
end

% Load audio and sampling frequency
y = app.AudioData;
Fs = app.Fs;

% Convert to mono if stereo
if size(y, 2) > 1
    y = mean(y, 2);
end

% FFT
N = length(y);
Y = fft(y);
f = (0:N-1)*(Fs/N);  % Full frequency range

% High-pass filter mask (symmetric)
cutoff = 1000; % Hz
H = ones(N, 1);  % Column vector
cut_idx = round(cutoff * N / Fs);

% Block low frequencies on both sides (symmetry)
H(1:cut_idx) = 0;
H(end - cut_idx + 1:end) = 0;

% Apply filter
Y_filtered = Y .* H;

% Inverse FFT to get time-domain signal
y_filtered = real(ifft(Y_filtered));

% Save and prepare for playback
app.filteredAudio = y_filtered;
app.Player = audioplayer(app.filteredAudio, app.Fs);

% Plot only positive half of frequency spectrum
f_half = f(1:floor(N/2)+1);
Yf_half = abs(Y_filtered(1:floor(N/2)+1));

plot(app.UIAxes3, f_half, Yf_half);
title(app.UIAxes3, 'High-Pass Filter Applied');
xlabel(app.UIAxes3, 'Frequency (Hz)');
ylabel(app.UIAxes3, '|Y(f)|');
        
        end

        % Button pushed function: LowPassButton
        function LowPassButtonPushed(app, event)
             if isempty(app.AudioData)
    uialert(app.UIFigure, 'Please upload audio first.', 'Error');
    return;
end

y = app.AudioData;
Fs = app.Fs;

% Convert to mono
if size(y, 2) > 1
    y = mean(y, 2);
end

N = length(y);
Y = fft(y);

% Frequency vector (positive half)
f = (0:floor(N/2)) * (Fs/N);

% Initialize filter mask
H = zeros(N, 1);

% Create low-pass mask for positive half
H_pos = double(f < 1000);  % logical to double (e.g. 1 or 0)

% Assign to H
H(1:floor(N/2)+1) = H_pos;

% Mirror for negative frequencies
H(end - floor(N/2) + 1:end) = flip(H_pos(2:end));  % skip DC to avoid duplicate

% Apply filter
Y_filtered = Y .* H;

% Inverse FFT
y_filtered = real(ifft(Y_filtered));

% Save and prepare playback
app.filteredAudio = y_filtered;
app.Player = audioplayer(app.filteredAudio, Fs);

% Plot (positive half)
Y_filtered_mag = abs(Y_filtered(1:floor(N/2)+1));
plot(app.UIAxes3, f, Y_filtered_mag);
title(app.UIAxes3, 'Low-Pass Filter Applied');
xlabel(app.UIAxes3, 'Frequency (Hz)');
ylabel(app.UIAxes3, '|Y(f)|');
xlim(app.UIAxes3, [0 Fs/2]);

        
        end

        % Button pushed function: BandStartButton
        function BandStartButtonPushed(app, event)
                if isempty(app.AudioData)
        uialert(app.UIFigure, 'Please upload audio first.', 'Error');
        return;
    end

    y = app.AudioData;
    Fs = app.Fs;

    if size(y, 2) > 1
        y = mean(y, 2);
    end

    N = length(y);
    Y = fft(y);
    f = (0:N-1)*(Fs/N);

    % Band-pass range
    f_low = 500;     % Lower bound (Hz)
    f_high = 3000;   % Upper bound (Hz)

    H = zeros(N, 1);
    H(f >= f_low & f <= f_high) = 1;

    Y_filtered = Y .* H;
    y_filtered = real(ifft(Y_filtered));

    app.filteredAudio = y_filtered;
    app.Player = audioplayer(app.filteredAudio, app.Fs);

    plot(app.UIAxes3, f, abs(Y_filtered));
    title(app.UIAxes3, 'Band Pass Filter Applied');
    xlabel(app.UIAxes3, 'Frequency (Hz)');
    ylabel(app.UIAxes3, '|Y(f)|');



        end

        % Button pushed function: BandStopButton
        function BandStopButtonPushed(app, event)
                if isempty(app.AudioData)
        uialert(app.UIFigure, 'Please upload audio first.', 'Error');
        return;
    end

    y = app.AudioData;
    Fs = app.Fs;

    if size(y, 2) > 1
        y = mean(y, 2);
    end

    N = length(y);
    Y = fft(y);
    f = (0:N-1)*(Fs/N);

    % Band-stop range
    f_low = 500;     % Lower bound (Hz)
    f_high = 3000;   % Upper bound (Hz)

    H = ones(N, 1);
    H(f >= f_low & f <= f_high) = 0;

    Y_filtered = Y .* H;
    y_filtered = real(ifft(Y_filtered));

    app.filteredAudio = y_filtered;
    app.Player = audioplayer(app.filteredAudio, app.Fs);

    plot(app.UIAxes3, f, abs(Y_filtered));
    title(app.UIAxes3, 'Band Stop Filter Applied');
    xlabel(app.UIAxes3, 'Frequency (Hz)');
    ylabel(app.UIAxes3, '|Y(f)|');
   
        end
    end

    % Component initialization
    methods (Access = private)

        % Create UIFigure and components
        function createComponents(app)

            % Create UIFigure and hide until all components are created
            app.UIFigure = uifigure('Visible', 'off');
            app.UIFigure.Color = [0 1 1];
            app.UIFigure.Position = [100 100 640 480];
            app.UIFigure.Name = 'MATLAB App';

            % Create UIAxes
            app.UIAxes = uiaxes(app.UIFigure);
            title(app.UIAxes, 'Time Domain')
            xlabel(app.UIAxes, 'X')
            ylabel(app.UIAxes, 'Y')
            zlabel(app.UIAxes, 'Z')
            app.UIAxes.Position = [24 218 300 185];

            % Create UIAxes2
            app.UIAxes2 = uiaxes(app.UIFigure);
            title(app.UIAxes2, 'Frequency Domain')
            xlabel(app.UIAxes2, 'X')
            ylabel(app.UIAxes2, 'Y')
            zlabel(app.UIAxes2, 'Z')
            app.UIAxes2.Position = [323 271 300 185];

            % Create UIAxes3
            app.UIAxes3 = uiaxes(app.UIFigure);
            title(app.UIAxes3, 'Filters')
            xlabel(app.UIAxes3, 'X')
            ylabel(app.UIAxes3, 'Y')
            zlabel(app.UIAxes3, 'Z')
            app.UIAxes3.Position = [331 63 300 185];

            % Create UploadAudioButton
            app.UploadAudioButton = uibutton(app.UIFigure, 'push');
            app.UploadAudioButton.ButtonPushedFcn = createCallbackFcn(app, @UploadAudioButtonPushed, true);
            app.UploadAudioButton.BackgroundColor = [0.3922 0.8314 0.0745];
            app.UploadAudioButton.FontWeight = 'bold';
            app.UploadAudioButton.Position = [65 184 100 23];
            app.UploadAudioButton.Text = 'Upload Audio';

            % Create StartStopButton
            app.StartStopButton = uibutton(app.UIFigure, 'push');
            app.StartStopButton.ButtonPushedFcn = createCallbackFcn(app, @StartStopButtonPushed, true);
            app.StartStopButton.BackgroundColor = [0.3922 0.8314 0.0745];
            app.StartStopButton.FontWeight = 'bold';
            app.StartStopButton.Position = [194 184 100 23];
            app.StartStopButton.Text = 'Start/Stop';

            % Create FadeInButton
            app.FadeInButton = uibutton(app.UIFigure, 'push');
            app.FadeInButton.ButtonPushedFcn = createCallbackFcn(app, @FadeInButtonPushed, true);
            app.FadeInButton.BackgroundColor = [1 0.4118 0.1608];
            app.FadeInButton.FontWeight = 'bold';
            app.FadeInButton.Position = [65 144 100 23];
            app.FadeInButton.Text = 'Fade In';

            % Create FadeOutButton
            app.FadeOutButton = uibutton(app.UIFigure, 'push');
            app.FadeOutButton.ButtonPushedFcn = createCallbackFcn(app, @FadeOutButtonPushed, true);
            app.FadeOutButton.BackgroundColor = [1 0.4118 0.1608];
            app.FadeOutButton.FontWeight = 'bold';
            app.FadeOutButton.Position = [194 144 100 23];
            app.FadeOutButton.Text = 'Fade Out';

            % Create Label
            app.Label = uilabel(app.UIFigure);
            app.Label.HorizontalAlignment = 'right';
            app.Label.FontWeight = 'bold';
            app.Label.Position = [49 103 47 22];
            app.Label.Text = 'Volume';

            % Create VolumeSlider
            app.VolumeSlider = uislider(app.UIFigure);
            app.VolumeSlider.ValueChangedFcn = createCallbackFcn(app, @VolumeSliderValueChanged, true);
            app.VolumeSlider.FontWeight = 'bold';
            app.VolumeSlider.Position = [117 112 150 3];

            % Create SpeedSliderLabel
            app.SpeedSliderLabel = uilabel(app.UIFigure);
            app.SpeedSliderLabel.HorizontalAlignment = 'right';
            app.SpeedSliderLabel.FontWeight = 'bold';
            app.SpeedSliderLabel.Position = [55 42 41 22];
            app.SpeedSliderLabel.Text = 'Speed';

            % Create SpeedSlider
            app.SpeedSlider = uislider(app.UIFigure);
            app.SpeedSlider.ValueChangedFcn = createCallbackFcn(app, @SpeedSliderValueChanged, true);
            app.SpeedSlider.FontWeight = 'bold';
            app.SpeedSlider.Position = [117 51 150 3];

            % Create AddFFTButton
            app.AddFFTButton = uibutton(app.UIFigure, 'push');
            app.AddFFTButton.ButtonPushedFcn = createCallbackFcn(app, @AddFFTButtonPushed, true);
            app.AddFFTButton.BackgroundColor = [1 1 0.0667];
            app.AddFFTButton.Position = [523 247 100 23];
            app.AddFFTButton.Text = 'Add FFT';

            % Create HighPassButton
            app.HighPassButton = uibutton(app.UIFigure, 'push');
            app.HighPassButton.ButtonPushedFcn = createCallbackFcn(app, @HighPassButtonPushed, true);
            app.HighPassButton.BackgroundColor = [1 1 0];
            app.HighPassButton.Position = [369 41 100 23];
            app.HighPassButton.Text = 'High Pass';

            % Create LowPassButton
            app.LowPassButton = uibutton(app.UIFigure, 'push');
            app.LowPassButton.ButtonPushedFcn = createCallbackFcn(app, @LowPassButtonPushed, true);
            app.LowPassButton.BackgroundColor = [1 1 0];
            app.LowPassButton.Position = [523 39 100 23];
            app.LowPassButton.Text = 'LowPass';

            % Create BandStartButton
            app.BandStartButton = uibutton(app.UIFigure, 'push');
            app.BandStartButton.ButtonPushedFcn = createCallbackFcn(app, @BandStartButtonPushed, true);
            app.BandStartButton.BackgroundColor = [1 1 0];
            app.BandStartButton.Position = [369 9 100 23];
            app.BandStartButton.Text = 'Band Start';

            % Create BandStopButton
            app.BandStopButton = uibutton(app.UIFigure, 'push');
            app.BandStopButton.ButtonPushedFcn = createCallbackFcn(app, @BandStopButtonPushed, true);
            app.BandStopButton.BackgroundColor = [1 1 0];
            app.BandStopButton.Position = [523 9 100 23];
            app.BandStopButton.Text = 'Band Stop';

            % Create InstrumentLabel
            app.InstrumentLabel = uilabel(app.UIFigure);
            app.InstrumentLabel.FontSize = 24;
            app.InstrumentLabel.Position = [65 424 118 32];
            app.InstrumentLabel.Text = 'Instrument';

            % Create Label_2
            app.Label_2 = uilabel(app.UIFigure);
            app.Label_2.HorizontalAlignment = 'right';
            app.Label_2.Position = [194 429 25 22];
            app.Label_2.Text = '';

            % Create Lamp
            app.Lamp = uilamp(app.UIFigure);
            app.Lamp.Position = [234 429 20 20];

            % Show the figure after all components are created
            app.UIFigure.Visible = 'on';
        end
    end

    % App creation and deletion
    methods (Access = public)

        % Construct app
        function app = appfinal

            % Create UIFigure and components
            createComponents(app)

            % Register the app with App Designer
            registerApp(app, app.UIFigure)

            if nargout == 0
                clear app
            end
        end

        % Code that executes before app deletion
        function delete(app)

            % Delete UIFigure when app is deleted
            delete(app.UIFigure)
        end
    end
end
