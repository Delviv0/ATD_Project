%% ================================================================
%  ANÁLISE E TRANSFORMAÇÃO DE DADOS – PROJETO 2026
%  Meta 1 – Pontos 1 ao 13
%
%  Estrutura esperada:
%    data/
%      01/
%        0_01_0.wav   <- digito=0, participante=01, repeticao=0
%        ...
%
%  IMPORTANTE: coloque este script na mesma pasta onde está a pasta data/
%  As funções locais estão no FINAL deste ficheiro (obrigatório MATLAB).
% ================================================================

% clear: apaga todas as variáveis do workspace
% clc: limpa a Command Window
% close all: fecha todas as figuras abertas
clear; clc; close all;

% Pasta do participante escolhido pelo grupo (alterar conforme necessário)
DATA_PATH = fullfile('data', '01');

%% ================================================================
%  PONTO 1 – Criação da Estrutura de Dados
% ================================================================
fprintf('=== PONTO 1: Criação da estrutura de dados ===\n');

if ~isfolder(DATA_PATH)
    error('Pasta "%s" não encontrada. Ajuste a variável DATA_PATH.', DATA_PATH);
end

% fullfile: constrói o caminho 'data/01/*.wav'
% dir: lista todos os ficheiros .wav que correspondem ao padrão
wav_files = dir(fullfile(DATA_PATH, '*.wav'));
fprintf('Ficheiros encontrados: %d\n', numel(wav_files));

directory    = {};
filename_col = {};
participant  = [];
digit_col    = [];
repetition   = [];

for f = 1:numel(wav_files)
    fname = wav_files(f).name;
    parts = strsplit(fname(1:end-4), '_');

    directory{end+1,1}    = DATA_PATH;
    filename_col{end+1,1} = fname;
    digit_col(end+1,1)    = str2double(parts{1});
    participant(end+1,1)  = str2double(parts{2});
    repetition(end+1,1)   = str2double(parts{3});
end

T = table(directory, filename_col, participant, digit_col, repetition, ...
    'VariableNames',{'Directory','Filename','Participant','Digit','Repetition'});
T = sortrows(T, {'Digit','Repetition'});

fprintf('Total de ficheiros  : %d\n',   height(T));
fprintf('Participante        : %d\n',   T.Participant(1));
fprintf('Dígitos             : %s\n',   num2str(unique(T.Digit)'));
fprintf('Repetições por díg. : %d\n\n', numel(unique(T.Repetition)));
disp(head(T, 5));
% 
%% ================================================================
%  PONTO 2 – Importação dos Sinais de Áudio
% ================================================================

N_files      = height(T);
signals      = cell(N_files, 1);
sample_rates = zeros(N_files, 1);

for i = 1:N_files
    filepath        = fullfile(T.Directory{i}, T.Filename{i});
    [sig, fs]       = audioread(filepath);
    signals{i}      = double(sig);
    sample_rates(i) = fs;
end

T.SampleRate = sample_rates;
T.Signal     = signals;

durations = cellfun(@numel, signals) ./ sample_rates;
fprintf('Taxa de amostragem : %d Hz\n',    T.SampleRate(1));
fprintf('Duração mínima     : %.3f s\n',   min(durations));
fprintf('Duração máxima     : %.3f s\n',   max(durations));
fprintf('Duração média      : %.3f s\n\n', mean(durations));

%% ================================================================
%  PONTO 3 – Representação Gráfica dos Sinais Originais
%  Exemplo: repetição 5
% ================================================================

REP_EXAMPLE = 5;

% Filtrar apenas a repetição escolhida e ordenar por dígito
mask   = (T.Repetition == REP_EXAMPLE);
subset = sortrows(T(mask,:), 'Digit');

figure('Name','Gráficos dos Áudios Originais', ...
       'NumberTitle','off', 'Position',[50 50 1100 900]);
sgtitle('Gráficos dos Áudios Originais', 'FontSize',13, 'FontWeight','bold');

for i = 1:height(subset)
    sig = subset.Signal{i};
    fs  = subset.SampleRate(i);
    t   = (0:numel(sig)-1) / fs;

    subplot(5, 2, i);
    plot(t, sig, 'Color',[0.18 0.45 0.69], 'LineWidth',0.5);
    title(sprintf('Dígito %d ; Repetição %d', subset.Digit(i), REP_EXAMPLE), ...
          'FontSize',9, 'FontWeight','bold');
    xlabel('Time [s]', 'FontSize',8);
    ylabel('Amplitude', 'FontSize',8);
    xlim([0, t(end)]); grid on;
end
% 
%% ================================================================
%  PONTO 4 – Pré-processamento dos Sinais
%   a) Remover silêncio inicial via energia por janelas
%   b) Normalizar amplitude para [-1, 1]
%   c) Uniformizar duração (padding/trimming)
% ================================================================

% O sinal é dividido em janelas de WINDOW_MS ms. Para cada janela calcula-se
% a energia. Se a energia estiver abaixo de ENERGY_THRESH, considera-se silêncio.
WINDOW_MS     = 1;       % tamanho de janela (ms)
ENERGY_THRESH = 0.008;   % limiar (fracção da energia máxima)

% Calcular duração alvo: percentil 95 das durações após remover silêncio
trimmed_lens = zeros(N_files, 1);
for i = 1:N_files
    s = remove_silence(T.Signal{i}, T.SampleRate(i), WINDOW_MS, ENERGY_THRESH);
    trimmed_lens(i) = numel(s);
end
target_s   = prctile(trimmed_lens ./ T.SampleRate, 95);
target_len = round(target_s * T.SampleRate(1));
fprintf('Duração alvo (percentil 95): %.4f s  (%d amostras)\n', target_s, target_len);

% Aplicar pré-processamento
preprocessed = cell(N_files, 1);
for i = 1:N_files
    sig = remove_silence(T.Signal{i}, T.SampleRate(i), WINDOW_MS, ENERGY_THRESH);
    sig = norm_amplitude(sig);
    sig = pad_trim(sig, target_len);
    preprocessed{i} = sig;
end

T.SignalPreprocessed = preprocessed;
% 
%% ================================================================
%  PONTO 5 – Gráficos dos Sinais Pré-processados
%  (repetição 5 — igual ao ponto 3)
% ================================================================

mask_pre   = (T.Repetition == REP_EXAMPLE);
subset_pre = sortrows(T(mask_pre,:), 'Digit');

figure('Name','Gráficos dos Áudios Pós-Processamento', ...
       'NumberTitle','off', 'Position',[50 50 1100 900]);
sgtitle('Gráficos dos Áudios Pós-Processamento', 'FontSize',13, 'FontWeight','bold');

for i = 1:height(subset_pre)
    sig = subset_pre.SignalPreprocessed{i};
    fs  = subset_pre.SampleRate(i);
    t   = (0:numel(sig)-1) / fs;

    subplot(5, 2, i);
    plot(t, sig, 'Color',[0.85 0.33 0.10], 'LineWidth',0.5);
    title(sprintf('Dígito %d ; Repetição %d', subset_pre.Digit(i), REP_EXAMPLE), ...
          'FontSize',9, 'FontWeight','bold');
    xlabel('Time [s]', 'FontSize',8);
    ylabel('Amplitude', 'FontSize',8);
    xlim([0, t(end)]); grid on;
end

% Guardar cada dígito como imagem individual de alta qualidade
for i = 1:height(subset_pre)
    sig = subset_pre.SignalPreprocessed{i};
    fs  = subset_pre.SampleRate(i);
    t   = (0:numel(sig)-1) / fs;

    fig = figure('Visible','off', 'Position',[100 100 900 400]);
    plot(t, sig, 'Color',[0.85 0.33 0.10], 'LineWidth',0.8);
    title(sprintf('Dígito %d ; Repetição %d', subset_pre.Digit(i), REP_EXAMPLE), ...
          'FontSize',12, 'FontWeight','bold');
    xlabel('Time [s]', 'FontSize',11);
    ylabel('Amplitude', 'FontSize',11);
    xlim([0, t(end)]); grid on;
    saveas(fig, sprintf('digit%d.png', subset_pre.Digit(i)));
    close(fig);
end
% 
%% ================================================================
%  PONTO 6 – Comparação Visual: Original vs Pré-processado
%  Grelha com os 10 dígitos lado a lado
% ================================================================

digits_list = sort(unique(T.Digit))';

figure('Name','Comparação Original vs Pré-processado', ...
       'NumberTitle','off', 'Position',[50 50 1300 1000]);
sgtitle('Comparação Original vs Pré-processado (Repetição 5)', ...
        'FontSize',12, 'FontWeight','bold');

for i = 1:numel(digits_list)
    d   = digits_list(i);
    idx = find(T.Digit==d & T.Repetition==REP_EXAMPLE, 1);

    fs    = T.SampleRate(idx);
    s_ori = T.Signal{idx};
    s_pre = T.SignalPreprocessed{idx};
    t_ori = (0:numel(s_ori)-1) / fs;
    t_pre = (0:numel(s_pre)-1) / fs;

    subplot(10, 2, 2*i-1);
    plot(t_ori, s_ori, 'Color',[0.18 0.45 0.69], 'LineWidth',0.5);
    title(sprintf('Dígito %d – Original', d), 'FontSize',8, 'FontWeight','bold');
    xlabel('Time [s]','FontSize',7); ylabel('Amplitude','FontSize',7);
    xlim([0, t_ori(end)]); grid on;

    subplot(10, 2, 2*i);
    plot(t_pre, s_pre, 'Color',[0.85 0.33 0.10], 'LineWidth',0.5);
    title(sprintf('Dígito %d – Pré-processado', d), 'FontSize',8, 'FontWeight','bold');
    xlabel('Time [s]','FontSize',7); ylabel('Amplitude','FontSize',7);
    xlim([0, t_pre(end)]); grid on;
end

fprintf(['Observações:\n' ...
    '- Silêncio inicial removido: todos os sinais começam no onset da fala.\n' ...
    '- Amplitude normalizada em [-1, 1]: elimina variações de ganho.\n' ...
    '- Duração uniforme: %d amostras (%.3f s) para todos os sinais.\n\n'], ...
    target_len, target_len / T.SampleRate(1));
% 
% %% ================================================================
% %  PONTO 7 – Características Temporais (7 features por áudio)
% %
% %  T1 – Energia total
% %  T2 – Amplitude máxima absoluta
% %  T3 – Desvio padrão da amplitude
% %  T4 – Zero-Crossing Rate (ZCR)
% %  T5 – Razão de energia: 1ª metade / 2ª metade
% %  T6 – Energia no onset (primeiros 25%)
% %  T7 – Root Mean Square (RMS)
% % ================================================================
% fprintf('=== PONTO 7: Cálculo de features temporais ===\n');
% 
% feat_energy_total = zeros(N_files, 1);
% feat_amp_max      = zeros(N_files, 1);
% feat_amp_std      = zeros(N_files, 1);
% feat_zcr          = zeros(N_files, 1);
% feat_energy_ratio = zeros(N_files, 1);
% feat_energy_onset = zeros(N_files, 1);
% feat_rms          = zeros(N_files, 1);
% 
% for i = 1:N_files
%     sig  = T.SignalPreprocessed{i};
%     n    = numel(sig);
%     half = floor(n / 2);
%     q1   = floor(n / 4);
% 
%     feat_energy_total(i)  = sum(sig .^ 2);
%     feat_amp_max(i)       = max(abs(sig));
%     feat_amp_std(i)       = std(sig);
%     feat_zcr(i)           = sum(abs(diff(sign(sig)))) / (2 * n);
% 
%     e1 = sum(sig(1:half) .^ 2) + 1e-12;
%     e2 = sum(sig(half+1:end) .^ 2) + 1e-12;
%     feat_energy_ratio(i)  = e1 / e2;
% 
%     feat_energy_onset(i)  = sum(sig(1:q1) .^ 2);
%     feat_rms(i)           = sqrt(mean(sig .^ 2));
% end
% 
% T.FeatEnergyTotal = feat_energy_total;
% T.FeatAmpMax      = feat_amp_max;
% T.FeatAmpStd      = feat_amp_std;
% T.FeatZCR         = feat_zcr;
% T.FeatEnergyRatio = feat_energy_ratio;
% T.FeatEnergyOnset = feat_energy_onset;
% T.FeatRMS         = feat_rms;
% 
% fprintf('  Features temporais calculadas para %d áudios.\n', N_files);
% fprintf('  Colunas adicionadas: FeatEnergyTotal, FeatAmpMax, FeatAmpStd,\n');
% fprintf('                       FeatZCR, FeatEnergyRatio, FeatEnergyOnset, FeatRMS\n\n');
% fprintf('Pontos 1 a 7 concluídos.\n');
% 
%% ================================================================
%  FUNÇÕES LOCAIS – OBRIGATORIAMENTE NO FINAL DO FICHEIRO
% ================================================================


function sig_out = remove_silence(sig, fs, window_ms, thresh)
% Remove o silêncio inicial com base na energia por janelas.
% Divide o sinal em janelas de window_ms ms e encontra a primeira
% janela cuja energia ultrapassa thresh*max_energia (onset da fala).
    % window_ms/1000 converte ms para segundos, *fs converte segundos para amostras
    win_samples = round(fs * window_ms / 1000);
    n_frames    = floor(numel(sig) / win_samples);

    energies = zeros(n_frames, 1);
    for k = 1:n_frames
        frame       = sig((k-1)*win_samples+1 : k*win_samples);
        energies(k) = sum(frame .^ 2);
    end

    max_e  = max(energies);
    onset  = find(energies / max_e > thresh, 1, 'first');
    sig_out = sig((onset-1)*win_samples+1 : end);
end
% 
% 
function sig_out = norm_amplitude(sig)
% Normaliza o sinal para [-1, 1] dividindo pelo máximo absoluto.
    mv      = max(abs(sig));
    sig_out = sig / mv;
end
% 
% 
function sig_out = pad_trim(sig, target_len)
% Se o sinal for maior que target_len, corta. Se for menor, adiciona zeros no final.
    if numel(sig) >= target_len
        sig_out = sig(1:target_len);
    else
        sig_out = [sig; zeros(target_len - numel(sig), 1)];
    end
end
