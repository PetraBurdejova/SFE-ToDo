
[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="880" alt="Visit QuantNet">](http://quantlet.de/index.php?p=info)

## [<img src="https://github.com/QuantLet/Styleguide-and-Validation-procedure/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **BaseCorrGaussModelCDO**[<img src="https://github.com/QuantLet/Styleguide-and-Validation-procedure/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/d3/ia)

```yaml
Name of QuantLet: BaseCorrGaussModelCDO

Published in: Statistics of Financial Markets

Description: 'Computes the base tranche correlation for a Gaussian credit default model.'

Keywords: CDO, Credit Risk, Equity tranche, Expected loss, Gauss Kronrod, Gaussian Model, bivariate, default, integration, multivariate normal, normal, numerical integration, correlation, gaussian, asset, financial, plot, graphical representation

See also: SFEbaseCorr, CompCorrGaussModelCDO, SFEPortfolioLossDensity, SFEcompCorr, SFEgaussCop, lowerTrLossGauss, SFEETLGaussTr1, ETL

Author: Awdesch Melzer

Author[Matlab]: Barbara Choros

Submitted: Wed, April 23 2014 by Awdesch Melzer

Submitted[Matlab]: Tue, May 17 2016 by Christoph Schult

Input: 
- a (scalar) : point from interval for optimization algorithm
- R (scalar) : recovery rate of security
- defProb (scalar) : default probability
- UAP (scalar) : upper attachment point of CDO tranche
- LAP (scalar) : 'lower attachment point of CDO tranche (also: detachment point)'
- DF (scalar) : discount factor
- DayCount (vector) : weights
- trueSpread (scalar) : true spread
- LowerTloss : Lower expecter tranche losses from lowerTrLossGauss routine

Output: 
- y : Base tranche correlation

Example: 'An example call is included in BaseCorrGaussModelCDO.R.'

Example[Matlab]: 'An example call for Matlab is included in ExampleCallBaseCorrGaussModelCDO.m'

```



### R Code:
```r

# clear variables and close windows
rm(list = ls(all = TRUE))
graphics.off()

# install and load packages
libraries = c("pracma")
lapply(libraries, function(x) if (!(x %in% installed.packages())) {
install.packages(x)
})
lapply(libraries, library, quietly = TRUE, character.only = TRUE)

bvnIntegrand = function(theta, b1, b2) {
    # Bivariate normal distribution | SUBROUNTINE of mvncdf() Integrand is 
    # exp(-(b1^2 + b2^2 - 2*b1*b2*sin(theta))/(2*cos(theta)^2) )
    sintheta = sin(theta)
    cossqtheta = cos(theta)^2  # always positive
    integrand = exp(-((b1 * sintheta - b2)^2/cossqtheta + b1^2)/2)
    return(integrand)
}

mvncdf = function(b, mu, sigma) {
    # MVNCDF Multivariate normal cumulative distribution function.  P = MVNCDF(B,
    # MU, SIGMA) returns the joint cumulative probability using numerical
    # integration. B is a vector of values, MU is the mean parameter vector, and
    # SIGMA is the covariance matrix.
    n = NROW(b)
    b = as.matrix(b)
    if (NCOL(b) != length(mu)) {
        stop("The first two inputs must be vectors of the same length.")
    }
    # Rho = sigma/(sqrt(diag(sigma))%*%t(sqrt(diag(sigma))))
    rho = sigma[2]
    if (rho > 0) {
        p1 = pnorm(apply(b, 2, min))
        p1[any(is.nan(b), 2)] = NaN
    } else {
        p1 = pnorm(b[, 1]) - pnorm(-b[, 2])
        p1[p1 < 0] = 0  # max would drop NaNs
    }
    if (abs(rho) < 1) {
        loLimit = asin(rho)
        hiLimit = sign(rho) * pi/2
        p2 = numeric()
        for (i in 1:n) {
            b1 = b[i, 1]
            b2 = b[i, 2]
            p2[i] = integral(function(x) bvnIntegrand(x, b1, b2), xmin = loLimit, 
                xmax = hiLimit, method = "Kronrod", reltol = 1e-10)
        }
    } else {
        p2 = rep(0, length(p1))
    }
    p = p1 - p2/(2 * pi)
    return(p)
}

BaseCorrGaussModelCDO = function(a, R, defProb, UAP, LAP, DF, DayCount, trueSpread, 
    LowerTLoss) {
    C = qnorm(defProb, 0, 1)
    NinvK = qnorm(UAP/(1 - R), 0, 1)
    A = (C - sqrt(1 - a^2) * NinvK)/a
    Sigma = matrix(c(1, -a, -a, 1), 2, 2)
    Mu = c(0, 0)
    EL1 = mvncdf(cbind(C, -A), Mu, Sigma)
    EL2 = pnorm(A)
    if (LAP == 0) {
        EL = EL1/UAP * (1 - R) + EL2
    } else {
        UpperETL = EL1 + EL2 * UAP/(1 - R)
        EL = (UpperETL - LowerTLoss)/(UAP - LAP) * (1 - R)
    }
    ProtectLeg = sum(diff(c(0, EL)) * DF)
    PremiumLeg = sum((1 - EL) * DF * DayCount)
    spread = ProtectLeg/PremiumLeg * 10000
    if (LAP == 0) {
        spread = (ProtectLeg - 0.05 * PremiumLeg) * 100
    }
    y = abs(spread - trueSpread)
    return(y)
}

# Example Call

# define input parameters
a          = 0.5;
trueSpread = 20.45;
recoveryR  = 0.4;
UAP        = 0.03;
LAP        = 0;
DF         = 0.9;
DayCount   = 0.25;
defProb    = 0.001;

# execute function
BaseCorrGaussModelCDO(a, recoveryR, defProb, UAP, LAP, DF, DayCount, trueSpread)
```
### Matlab Code
```matlab
% define function to compute the base correlation of a gaussian model
function y = BaseCorrGaussModelCDO(a, R, defProb, UAP, LAP, DF, DayCount, trueSpread, LowerTLoss)
    C     = norminv(defProb, 0, 1);
    NinvK = norminv(UAP / (1 - R), 0, 1);
    A     = (C - sqrt(1 - a^2) * NinvK) / a;
    Sigma = [1 -a; -a 1];
    Mu    = [0 0];
    EL1   = mvncdf([C, -A], Mu, Sigma);
    EL2   = normcdf(A);

    if LAP == 0
        EL       = EL1 / UAP * (1 - R) + EL2;
    else
        UpperETL = EL1 + EL2 * UAP/(1 - R);
        EL       = (UpperETL - LowerTLoss) / (UAP - LAP) * (1 - R);
    end

    ProtectLeg = sum(diff([0; EL]) .* DF);
    PremiumLeg = sum((1 - EL) .* DF .* DayCount);
    spread     = ProtectLeg / PremiumLeg * 10000;

    if LAP == 0
        spread = (ProtectLeg - 0.05 * PremiumLeg) * 100;
    end

    y = abs(spread - trueSpread);
```
