%% ============================================================
% IEEE SPS Bangalore + MathWorks Challenge
% Motor Imagery EEG Classification
%
% PART 1 : Dataset Loading
% ============================================================

clc;
clear;
close all;

fprintf('=============================================\n');
fprintf(' Motor Imagery EEG Classification\n');
fprintf(' IEEE SPS Bangalore + MathWorks\n');
fprintf('=============================================\n\n');

% ============================================================
% Select Dataset Folder
% ============================================================

datasetFolder = uigetdir(pwd,'Select Folder containing MAT files');

if isequal(datasetFolder,0)
    error('No folder selected.');
end

fprintf('Dataset Folder : %s\n\n',datasetFolder);

% ============================================================
% Search MAT Files
% ============================================================

files = dir(fullfile(datasetFolder,'*.mat'));

fprintf('Total MAT Files Found : %d\n\n',length(files));

if isempty(files)
    error('No MAT files found.');
end

% ============================================================
% Store Complete File Paths
% ============================================================

filePaths = strings(length(files),1);

for i = 1:length(files)

    filePaths(i) = fullfile(files(i).folder,files(i).name);

end

disp('First Five Files Found :')

disp(filePaths(1:min(5,end)))

fprintf('\n');

% ============================================================
% Create Empty Containers
% ============================================================

EEGData = cell(length(files),1);

Labels = {};

Session = [];

Subject = [];

fprintf('Containers Created Successfully.\n\n');

% ============================================================
% Load Dataset
% ============================================================

fprintf('Loading Dataset...\n\n');

for i = 1:length(filePaths)

    fprintf('[%d/%d] %s\n',i,length(filePaths),files(i).name);

    EEGData{i} = load(filePaths(i));

end

fprintf('\n=====================================\n');
fprintf('Dataset Loaded Successfully\n');
fprintf('=====================================\n\n');

fprintf('Total Files Loaded : %d\n\n',length(EEGData));

% ============================================================
% Inspect First MAT File
% ============================================================

disp('Variables Present in First MAT File :')

vars = fieldnames(EEGData{1})

% ============================================================
% Display Information About Each Variable
% ============================================================

disp('--------------------------------------------')
disp('Variable Information')
disp('--------------------------------------------')

for k = 1:length(vars)

    value = EEGData{1}.(vars{k});

    fprintf('\nVariable %d : %s\n',k,vars{k});

    disp(class(value));

    disp(size(value));

end

disp('--------------------------------------------')

fprintf('\nPart 1 Completed Successfully.\n');

%% %% ============================================================
% PART 2
% Dataset Inspection
% ============================================================

fprintf('\n');
fprintf('=========================================\n');
fprintf('PART 2 : DATASET INSPECTION\n');
fprintf('=========================================\n\n');

TotalLoaded = length(EEGData);

fprintf('Total Successfully Loaded Files : %d\n\n',TotalLoaded);

Inspection = struct();

for i = 1:TotalLoaded

    fprintf('Inspecting File %d of %d\n',i,TotalLoaded);

    data = EEGData{i};

    %------------------------------------------------------
    % Signal
    %------------------------------------------------------

    if isfield(data,'signal')

        signal = data.signal;

        Inspection(i).Samples = size(signal,1);

        Inspection(i).Channels = size(signal,2);

    else

        Inspection(i).Samples = NaN;

        Inspection(i).Channels = NaN;

    end

    %------------------------------------------------------
    % Sampling Rate
    %------------------------------------------------------

    if isfield(data,'SampleRate')

        Inspection(i).Fs = double(data.SampleRate);

    else

        Inspection(i).Fs = NaN;

    end

    %------------------------------------------------------
    % Event Markers
    %------------------------------------------------------

    if isfield(data,'mark')

        Inspection(i).Trials = size(data.mark,1);

    else

        Inspection(i).Trials = 0;

    end

    %------------------------------------------------------
    % Display Information
    %------------------------------------------------------

    fprintf('Samples       : %d\n',Inspection(i).Samples);

    fprintf('Channels      : %d\n',Inspection(i).Channels);

    fprintf('Sampling Rate : %d Hz\n',Inspection(i).Fs);

    fprintf('Trials        : %d\n\n',Inspection(i).Trials);

end

fprintf('Inspection Completed Successfully.\n');

%% %% ============================================================
% IEEE SPS Bangalore + MathWorks Challenge
% PART 3 : EEG Preprocessing
% ============================================================

fprintf('=========================================\n');
fprintf(' PART 3 : EEG PREPROCESSING\n');
fprintf('=========================================\n\n');

% ------------------------------------------------------------
% Select MAT Dataset Folder
% ------------------------------------------------------------

datasetFolder = uigetdir(pwd,'Select MAT Dataset Folder');

if isequal(datasetFolder,0)
    error('No folder selected.');
end

files = dir(fullfile(datasetFolder,'*.mat'));

fprintf("MAT Files Found : %d\n\n",length(files));

% ------------------------------------------------------------
% Create Cell Arrays
% ------------------------------------------------------------

ProcessedEEG = {};
Labels = {};

validCount = 0;

% ------------------------------------------------------------
% Filter Design
% ------------------------------------------------------------

Fs = 250;              % Default (will update if file contains SampleRate)

bpFilt = designfilt( ...
    'bandpassiir', ...
    'FilterOrder',6,...
    'HalfPowerFrequency1',8,...
    'HalfPowerFrequency2',30,...
    'SampleRate',Fs);

notchFilt = designfilt( ...
    'bandstopiir', ...
    'FilterOrder',2,...
    'HalfPowerFrequency1',49,...
    'HalfPowerFrequency2',51,...
    'SampleRate',Fs);

% ------------------------------------------------------------
% Loop through MAT files
% ------------------------------------------------------------

for i = 1:length(files)

    filename = fullfile(files(i).folder,files(i).name);

    fprintf("[%d/%d] %s\n",i,length(files),files(i).name);

    try

        data = load(filename);

        % ---------------------------------------
        % Check variables
        % ---------------------------------------

        if ~isfield(data,'signal')
            fprintf("Signal not found.\n\n");
            continue;
        end

        eeg = double(data.signal);

        % ---------------------------------------
        % Update Sample Rate
        % ---------------------------------------

        if isfield(data,'SampleRate')
            Fs = double(data.SampleRate);

            bpFilt = designfilt( ...
                'bandpassiir', ...
                'FilterOrder',6,...
                'HalfPowerFrequency1',8,...
                'HalfPowerFrequency2',30,...
                'SampleRate',Fs);

            notchFilt = designfilt( ...
                'bandstopiir', ...
                'FilterOrder',2,...
                'HalfPowerFrequency1',49,...
                'HalfPowerFrequency2',51,...
                'SampleRate',Fs);
        end

        % ---------------------------------------
        % Keep only EEG channels
        % ---------------------------------------

        if size(eeg,2) >= 32
            eeg = eeg(:,1:32);
        end

        % ---------------------------------------
        % Remove Mean
        % ---------------------------------------

        eeg = eeg - mean(eeg);

        % ---------------------------------------
        % Bandpass Filter
        % ---------------------------------------

        eeg = filtfilt(bpFilt,eeg);

        % ---------------------------------------
        % Notch Filter
        % ---------------------------------------

        eeg = filtfilt(notchFilt,eeg);

        % ---------------------------------------
        % Normalize
        % ---------------------------------------

        eeg = normalize(eeg);

        % ---------------------------------------
        % Store
        % ---------------------------------------

        validCount = validCount + 1;

        ProcessedEEG{validCount} = eeg;

        if isfield(data,'mark')
            Labels{validCount} = data.mark;
        else
            Labels{validCount} = [];
        end

    catch ME

        fprintf("Skipped : %s\n",ME.message);

    end

end

% ------------------------------------------------------------
% Remove Empty Cells
% ------------------------------------------------------------

ProcessedEEG = ProcessedEEG(1:validCount);
Labels = Labels(1:validCount);

fprintf('\n=====================================\n');
fprintf('Preprocessing Completed\n');
fprintf('=====================================\n\n');

fprintf('Valid EEG Files : %d\n\n',validCount);

% ------------------------------------------------------------
% Visualize First Recording
% ------------------------------------------------------------

figure;

plot(ProcessedEEG{1}(:,1));

xlabel('Samples');

ylabel('Amplitude');

title('Filtered EEG (Channel 1)');

grid on;


%% ==========================================================
% PART 4 : Improved Epoch Extraction
% Motor Imagery Window (2 - 6 sec)
% ===========================================================

% Folder containing all MAT files

datasetFolder = pwd;

files = dir(fullfile(datasetFolder,'**','*.mat'));

fprintf("MAT Files Found : %d\n\n",length(files));

% Parameters

Fs = 250;                  % Sampling Frequency
startTime = 2;             % seconds after cue
endTime   = 6;             % seconds after cue

startOffset = round(startTime*Fs);
endOffset   = round(endTime*Fs);

samplesPerEpoch = endOffset-startOffset;

Epochs = {};
Labels = [];

% ==========================================================
% Read all files
% ==========================================================

for f = 1:length(files)

    fprintf("[%d/%d] %s\n",...
        f,length(files),files(f).name);

    S = load(fullfile(files(f).folder,files(f).name));

    vars = fieldnames(S);

    EEG = [];
    events = [];

    % Automatically detect EEG matrix

    for k=1:length(vars)

        value = S.(vars{k});

        if isnumeric(value)

            if size(value,2)>=29 && size(value,1)>1000

                EEG = value(:,1:29);

                break;

            end

        end

    end

    % Automatically detect event matrix

    for k=1:length(vars)

        value = S.(vars{k});

        if isnumeric(value)

            if size(value,2)>=2 && size(value,1)<500

                events = value;

                break;

            end

        end

    end

    if isempty(EEG) || isempty(events)
        continue;
    end

    % =======================================================
    % Extract Epochs
    % =======================================================

    for e=1:size(events,1)

        eventPos = round(events(e,1));
        eventCode = events(e,2);

        startSample = eventPos + startOffset;
        endSample   = eventPos + endOffset - 1;

        if startSample<1
            continue;
        end

        if endSample>size(EEG,1)
            continue;
        end

        epoch = EEG(startSample:endSample,:);

        Epochs{end+1} = epoch;
        Labels(end+1,1) = eventCode;

    end

end

% Convert Labels

Labels = double(Labels);

% Display Summary

fprintf("\n");
fprintf("=========================================\n");
fprintf("Part 4 Summary\n");
fprintf("=========================================\n");

fprintf("Total Epochs      : %d\n",length(Epochs));
fprintf("Samples / Epoch   : %d\n",size(Epochs{1},1));
fprintf("Channels          : %d\n",size(Epochs{1},2));

fprintf("\nUnique Labels\n");
disp(unique(Labels));

% Plot first epoch

figure

plot(Epochs{1}(:,1),'LineWidth',1.2)

xlabel('Samples')
ylabel('Amplitude')
title('First EEG Epoch')
grid on

disp(" ");
disp("Part 4 Completed Successfully.");

%% ==========================================================
% PART 5 : Advanced EEG Feature Extraction
% ==========================================================

Fs = 250;

numEpochs = length(Epochs);
numChannels = size(Epochs{1},2);

fprintf("Epochs    : %d\n",numEpochs);
fprintf("Channels  : %d\n\n",numChannels);

% Feature Matrix

Features = [];

% ----------------------------------------------------------
% Feature Extraction
% ----------------------------------------------------------

for e = 1:numEpochs

    epoch = Epochs{e};

    featureVector = [];

    for ch = 1:numChannels

        x = double(epoch(:,ch));

        % -------------------------------
        % Time Domain Features
        % -------------------------------

        meanVal = mean(x);

        stdVal = std(x);

        varVal = var(x);

        rmsVal = rms(x);

        % Hjorth Parameters

        dx = diff(x);

        ddx = diff(dx);

        activity = varVal;

        mobility = sqrt(var(dx)/var(x));

        complexity = sqrt(var(ddx)/var(dx))/mobility;

        % -------------------------------
        % Frequency Domain
        % -------------------------------

        [Pxx,F] = pwelch(x,[],[],[],Fs);

        delta = bandpower(Pxx,F,[0.5 4],'psd');

        theta = bandpower(Pxx,F,[4 8],'psd');

        alpha = bandpower(Pxx,F,[8 13],'psd');

        beta = bandpower(Pxx,F,[13 30],'psd');

        % Spectral Entropy

        P = Pxx/sum(Pxx);

        entropyVal = -sum(P.*log2(P+eps));

        % Feature Vector

        featureVector = [featureVector ...
            meanVal ...
            stdVal ...
            varVal ...
            rmsVal ...
            activity ...
            mobility ...
            complexity ...
            delta ...
            theta ...
            alpha ...
            beta ...
            entropyVal];

    end

    Features(e,:) = featureVector;

    if mod(e,250)==0

        fprintf("%d / %d Epochs Processed\n",e,numEpochs);

    end

end

% ----------------------------------------------------------
% Remove NaN / Inf
% ----------------------------------------------------------

Features(isnan(Features)) = 0;

Features(isinf(Features)) = 0;

% ----------------------------------------------------------
% Normalize
% ----------------------------------------------------------

Features = normalize(Features);

% ----------------------------------------------------------
% Summary
% ----------------------------------------------------------

disp(" ");

disp("=========================================");
disp("Feature Extraction Completed");
disp("=========================================");

fprintf("Epochs          : %d\n",numEpochs);
fprintf("Channels        : %d\n",numChannels);
fprintf("Features/Epoch  : %d\n",size(Features,2));

disp(" ");

disp("Feature Matrix Size");
disp(size(Features));

disp(" ");

disp("Part 5 Completed Successfully.");

%% ==========================================================
% PART 6 : Dataset Preparation (Improved)
% ==========================================================

disp("=========================================");
disp("PART 6 : DATASET PREPARATION");
disp("=========================================");

% ----------------------------------------------------------
% Check Variables
% ----------------------------------------------------------

if ~exist('Features','var') || ~exist('Labels','var')

    error(['Run Part-5 first so that Features and Labels ' ...
           'are available in the workspace.']);

end

% ----------------------------------------------------------
% Original Dataset
% ----------------------------------------------------------

fprintf("\nOriginal Dataset Size : %d x %d\n", ...
        size(Features,1), size(Features,2));

fprintf("Original Labels       : %d\n\n", length(Labels));

% ----------------------------------------------------------
% Keep Only Motor Imagery Classes
% ----------------------------------------------------------

validLabels = [769 770 771];

idx = ismember(Labels,validLabels);

Features = Features(idx,:);
Labels   = Labels(idx);

fprintf("After Removing Non-Motor Imagery Events\n");
fprintf("Remaining Samples : %d\n",size(Features,1));
fprintf("Feature Dimension : %d\n\n",size(Features,2));

% ----------------------------------------------------------
% Convert Labels
% ----------------------------------------------------------

newLabels = zeros(size(Labels));

newLabels(Labels==769)=0;
newLabels(Labels==770)=1;
newLabels(Labels==771)=2;

Labels = newLabels;

disp("Converted Labels")
disp(unique(Labels))

% ----------------------------------------------------------
% Remove NaN / Inf
% ----------------------------------------------------------

Features(isnan(Features)) = 0;
Features(isinf(Features)) = 0;

% ----------------------------------------------------------
% Normalize Features
% ----------------------------------------------------------

disp(" ");
disp("Normalizing Features...");

Features = normalize(Features,'zscore');

% ----------------------------------------------------------
% Shuffle Dataset
% ----------------------------------------------------------

rng(1);

idx = randperm(length(Labels));

Features = Features(idx,:);
Labels   = Labels(idx);

% ----------------------------------------------------------
% Stratified Train-Test Split
% ----------------------------------------------------------

cv = cvpartition(Labels,'HoldOut',0.20);

XTrain = Features(training(cv),:);
YTrain = Labels(training(cv));

XTest = Features(test(cv),:);
YTest = Labels(test(cv));

% ----------------------------------------------------------
% Dataset Summary
% ----------------------------------------------------------

disp(" ");
disp("=========================================");
disp("Dataset Summary");
disp("=========================================");

fprintf("Training Samples : %d\n",size(XTrain,1));
fprintf("Testing Samples  : %d\n",size(XTest,1));
fprintf("Features         : %d\n\n",size(XTrain,2));

% ----------------------------------------------------------
% Training Distribution
% ----------------------------------------------------------

disp("Training Label Distribution")

classes = unique(YTrain);

for i = 1:length(classes)

    count = sum(YTrain==classes(i));

    fprintf("Class %d : %4d samples (%.2f%%)\n",...
        classes(i),...
        count,...
        100*count/length(YTrain));

end

disp(" ");

disp("Testing Label Distribution")

classes = unique(YTest);

for i = 1:length(classes)

    count = sum(YTest==classes(i));

    fprintf("Class %d : %4d samples (%.2f%%)\n",...
        classes(i),...
        count,...
        100*count/length(YTest));

end

disp(" ");
disp("Part 6 Completed Successfully.");

%% ==========================================================
% PART 7 : ROBUST MULTI CLASS SVM CLASSIFICATION
% ==========================================================

disp("=========================================");
disp("PART 7 : IMPROVED SVM CLASSIFICATION");
disp("=========================================");


% Check Data

if ~exist('XTrain','var')

    error('Run Feature Extraction Part First');

end


fprintf("Training Data Size : %d x %d\n",...
size(XTrain,1),size(XTrain,2));


fprintf("Testing Data Size : %d x %d\n",...
size(XTest,1),size(XTest,2));


% Normalize Features

mu = mean(XTrain);

sigma = std(XTrain)+eps;


XTrain = (XTrain-mu)./sigma;

XTest  = (XTest-mu)./sigma;



% PCA Dimension Reduction


disp("Applying PCA...");


[coeff,score,~,~,explained]=pca(XTrain);


variance=sum(explained);


components=find(cumsum(explained)>=95,1);


XTrain=score(:,1:components);


XTest=(XTest-mu)*coeff(:,1:components);



fprintf("PCA Features Used : %d\n",components);



% Class Distribution


disp("Training Class Distribution")

tabulate(YTrain)



% Balanced SVM


disp("Training Optimized Balanced SVM...");


template = templateSVM(...
    'KernelFunction','rbf',...
    'KernelScale','auto',...
    'BoxConstraint',10,...
    'Standardize',true);



model = fitcecoc(...
    XTrain,...
    YTrain,...
    'Learners',template,...
    'Coding','onevsone');



disp("Training Completed");



% Prediction


YPred=predict(model,XTest);



accuracy=sum(YPred==YTest)/length(YTest)*100;


fprintf("\nAccuracy = %.2f %%\n",accuracy);



% Confusion Matrix


classes=unique(YTest);


CM=confusionmat(YTest,YPred);



figure

confusionchart(CM,string(classes));

title(sprintf("SVM Confusion Matrix Accuracy %.2f%%",accuracy));



disp("Confusion Matrix");

disp(CM);


% Precision Recall F1


numClasses=size(CM,1);


precision=zeros(numClasses,1);
recall=zeros(numClasses,1);
f1=zeros(numClasses,1);



for i=1:numClasses


    TP=CM(i,i);

    FP=sum(CM(:,i))-TP;

    FN=sum(CM(i,:))-TP;


    precision(i)=TP/(TP+FP+eps);

    recall(i)=TP/(TP+FN+eps);

    f1(i)=2*precision(i)*recall(i)/(precision(i)+recall(i)+eps);


end



Results=table(classes,...
precision,...
recall,...
f1,...
'VariableNames',...
{'Class','Precision','Recall','F1Score'});


disp(Results);



fprintf("\nAverage Precision %.3f\n",mean(precision));
fprintf("Average Recall %.3f\n",mean(recall));
fprintf("Average F1 %.3f\n",mean(f1));



% Cross Validation


disp("Cross Validation");


CV=crossval(model,...
'KFold',5);


loss=kfoldLoss(CV);


fprintf("CV Accuracy %.2f %%\n",(1-loss)*100);



disp("=========================================");
disp("PART 7 COMPLETED");
disp("=========================================");

%% ==========================================================
% PART 8 : ROBUST CSP FEATURE EXTRACTION
% ==========================================================


disp("=========================================");
disp("PART 8 : CSP FEATURE EXTRACTION");
disp("=========================================");



% Check Epochs


if ~exist('Epochs','var')

error("Epochs variable missing");

end


if isempty(Epochs)

error("Epochs is empty. Check epoch extraction");

end



fprintf("Total Epochs : %d\n",length(Epochs));



% Check Labels


if ~exist('Labels','var')

error("Labels missing");

end



fprintf("Available Labels:\n");

disp(unique(Labels));



% Automatically Remove Invalid Labels


valid=find(~isnan(Labels));


Epochs=Epochs(valid);

Labels=Labels(valid);



fprintf("Valid Epochs : %d\n",length(Epochs));



% Convert Labels to 0 1 2


classes=unique(Labels);


if length(classes)<2

error("Only one class found. Need multiple MI classes");

end


newLabels=zeros(size(Labels));


for i=1:length(classes)

newLabels(Labels==classes(i))=i-1;

end


Labels=newLabels;



fprintf("Classes detected : ");

disp(unique(Labels));



% EEG Dimension


sampleEpoch=Epochs{1};


[numSamples,numChannels]=size(sampleEpoch);



fprintf("Channels : %d\n",numChannels);
fprintf("Samples  : %d\n",numSamples);



% Bandpass Filter


Fs=250;


bp=designfilt(...
'bandpassiir',...
'FilterOrder',4,...
'HalfPowerFrequency1',8,...
'HalfPowerFrequency2',30,...
'SampleRate',Fs);



for i=1:length(Epochs)


X=Epochs{i};


for ch=1:numChannels

X(:,ch)=filtfilt(bp,X(:,ch));

end


Epochs{i}=X;


end



disp("Filtering Completed");



% CSP


numFilters=6;


CSPFeatures=[];


classList=unique(Labels);



for c=1:length(classList)


cls=classList(c);


classData=Epochs(Labels==cls);


otherData=Epochs(Labels~=cls);



C1=zeros(numChannels);

C2=zeros(numChannels);



for i=1:length(classData)

X=classData{i};

X=X';

covX=(X*X')/trace(X*X');

C1=C1+covX;

end


C1=C1/length(classData);



for i=1:length(otherData)

X=otherData{i};

X=X';

covX=(X*X')/trace(X*X');

C2=C2+covX;

end


C2=C2/length(otherData);



[V,D]=eig(C1,C1+C2);


[~,ind]=sort(diag(D),'descend');


V=V(:,ind);



filters=[V(:,1:3) V(:,end-2:end)];



features=zeros(length(Epochs),numFilters);



for e=1:length(Epochs)


X=Epochs{e}';


Z=filters'*X;


variance=var(Z,0,2);


features(e,:)=log(variance/sum(variance));


end



CSPFeatures=[CSPFeatures features];


end



% Normalize


CSPFeatures=zscore(CSPFeatures);



% Split


cv=cvpartition(Labels,'Holdout',0.2);


XCSPTrain=CSPFeatures(training(cv),:);

YCSPTrain=Labels(training(cv));


XCSPTest=CSPFeatures(test(cv),:);

YCSPTest=Labels(test(cv));



fprintf("\nCSP Feature Matrix\n");

fprintf("Samples : %d\n",size(CSPFeatures,1));

fprintf("Features: %d\n",size(CSPFeatures,2));



save CSP_Features.mat CSPFeatures Labels



disp("=========================================");
disp("PART 8 COMPLETED");
disp("=========================================");


%% ==========================================================
% PART 9 : Final CSP + Optimized SVM Classification
% ==========================================================

disp("=========================================");
disp("PART 9 : FINAL CSP CLASSIFIER");
disp("=========================================");

% ----------------------------------------------------------
% Check Variables
% ----------------------------------------------------------

if ~exist('XCSPTrain','var') || ~exist('XCSPTest','var')
    error('Run Part-8 first.');
end

% ----------------------------------------------------------
% Dataset Summary
% ----------------------------------------------------------

fprintf("\nTraining Samples : %d x %d\n", ...
    size(XCSPTrain,1), size(XCSPTrain,2));

fprintf("Testing Samples  : %d x %d\n\n", ...
    size(XCSPTest,1), size(XCSPTest,2));

% ----------------------------------------------------------
% Train Optimized ECOC SVM
% ----------------------------------------------------------

disp("Training Optimized ECOC Multi-class SVM...");

template = templateSVM(...
    'KernelFunction','rbf',...
    'KernelScale','auto',...
    'BoxConstraint',10,...
    'Standardize',true);

model = fitcecoc(...
    XCSPTrain,...
    YCSPTrain,...
    'Learners',template,...
    'Coding','onevsone');

disp("Training Completed.");

% ----------------------------------------------------------
% Prediction
% ----------------------------------------------------------

disp("Predicting Test Data...");

[YPred,score] = predict(model,XCSPTest);

% ----------------------------------------------------------
% Accuracy
% ----------------------------------------------------------

accuracy = mean(YPred==YCSPTest)*100;

fprintf("\n====================================\n");
fprintf("Overall Accuracy = %.2f %%\n",accuracy);
fprintf("====================================\n\n");

% ----------------------------------------------------------
% Confusion Matrix
% ----------------------------------------------------------

CM = confusionmat(YCSPTest,YPred);

disp("Confusion Matrix");

disp(CM);

figure;

confusionchart(CM,...
    {'Left','Right','Foot'},...
    'Title',...
    sprintf('Confusion Matrix (%.2f%% Accuracy)',accuracy));

% ----------------------------------------------------------
% Precision Recall F1
% ----------------------------------------------------------

numClasses = size(CM,1);

precision = zeros(numClasses,1);
recall = zeros(numClasses,1);
f1 = zeros(numClasses,1);

for i = 1:numClasses

    TP = CM(i,i);

    FP = sum(CM(:,i)) - TP;

    FN = sum(CM(i,:)) - TP;

    precision(i) = TP/(TP+FP+eps);

    recall(i) = TP/(TP+FN+eps);

    f1(i) = 2*precision(i)*recall(i)/(precision(i)+recall(i)+eps);

end

Results = table((0:2)',...
    precision,...
    recall,...
    f1,...
    'VariableNames',...
    {'Class','Precision','Recall','F1Score'});

disp(" ");

disp("Class-wise Performance");

disp(Results);

fprintf("\nAverage Precision : %.4f\n",mean(precision));

fprintf("Average Recall    : %.4f\n",mean(recall));

fprintf("Average F1 Score  : %.4f\n",mean(f1));

% ----------------------------------------------------------
% ROC Curves
% ----------------------------------------------------------

figure;

hold on

colors = lines(3);

legendText = cell(3,1);

for i = 1:3

    [Xroc,Yroc,~,AUC] = perfcurve(...
        YCSPTest,...
        score(:,i),...
        i-1);

    plot(Xroc,...
        Yroc,...
        'LineWidth',2,...
        'Color',colors(i,:));

    legendText{i} = sprintf(...
        'Class %d (AUC = %.3f)',...
        i-1,...
        AUC);

end

plot([0 1],[0 1],'k--');

xlabel('False Positive Rate');

ylabel('True Positive Rate');

title('ROC Curves');

legend(legendText,...
    'Location','SouthEast');

grid on

% ----------------------------------------------------------
% 5-Fold Cross Validation
% ----------------------------------------------------------

disp(" ");

disp("Performing Cross Validation...");

cvModel = crossval(model,'KFold',5);

cvAccuracy = (1-kfoldLoss(cvModel))*100;

fprintf("Cross Validation Accuracy = %.2f %%\n",...
    cvAccuracy);

% ----------------------------------------------------------
% Support Vector Information
% ----------------------------------------------------------

BinaryModels = model.BinaryLearners;

totalSV = 0;

for i = 1:length(BinaryModels)

    totalSV = totalSV + ...
        size(BinaryModels{i}.SupportVectors,1);

end

fprintf("Total Support Vectors : %d\n",...
    totalSV);

% ----------------------------------------------------------
% Per-Class Accuracy
% ----------------------------------------------------------

disp(" ");

disp("Per-Class Accuracy");

for i = 0:2

    idx = (YCSPTest==i);

    acc = mean(YPred(idx)==i)*100;

    fprintf("Class %d : %.2f %%\n",i,acc);

end

% ----------------------------------------------------------
% Finished
% ----------------------------------------------------------

disp(" ");

disp("=========================================");
disp("PART 9 COMPLETED SUCCESSFULLY");
disp("=========================================");


figure
scatter(CSPFeatures(:,1),...
CSPFeatures(:,2),...
50,...
Labels,...
'filled')
xlabel("CSP Feature 1")
ylabel("CSP Feature 2") 
title("CSP Feature Distribution")
colorbar
grid on
